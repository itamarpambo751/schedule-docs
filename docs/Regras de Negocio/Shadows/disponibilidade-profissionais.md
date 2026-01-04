# Disponibilidade Multi-Hospital & Multi-Especialidade — Modelo, Regras e Boas Práticas

Este documento descreve como modelar e aplicar **disponibilidade de profissionais** que trabalham em **múltiplos hospitais** e com **especialidades diferentes** em cada local, garantindo coerência com **Agendas (Schedules)** e **Slots**.

> **Ponto-chave:** um profissional **só entra** nos Slots de um dia se houver **interseção real** entre a janela da Agenda e o **WorkTime** (turno) desse profissional **na mesma unidade + especialidade + dia da semana** (e tipo de serviço compatível).

---

## 1) O que o teu esquema já cobre

- **Multi-hospital**: `ProfessionalWorkTime.healthUnitTaxId` diferencia turnos por unidade.
- **Multi-especialidade por hospital**: `ProfessionalWorkTime.specialityId` indica a especialidade activa naquele turno/local.
- **Janelas por local**: `weekDay`, `startAt`, `endsAt`, `validFrom/validTo`, `typeOfService` permitem descrever turnos distintos (ex.: *Cardiologia* no Hospital A às segundas; *Ortopedia* no Hospital B às terças).

---

## 2) Lacunas & Complementos Recomendados

### 2.1. Vínculo formal Profissional ↔ Unidade ↔ Especialidade

Além de `ProfessionalWorkTime`, recomenda-se um vínculo “contratual” explícito para validar que o profissional **pode** exercer a especialidade naquela unidade:

```prisma
model ProfessionalAssignment {
  id                String   @id @default(uuid())
  professionalTaxId String
  healthUnitTaxId   String
  specialityId      String
  typeOfService     TypeOfService?
  validFrom         DateTime? @db.Timestamptz
  validTo           DateTime? @db.Timestamptz
  isActive          Boolean   @default(true)

  // Diagnóstico/ETL
  sourceSystem String  @default("users-api")
  externalHash String?

  @@unique([professionalTaxId, healthUnitTaxId, specialityId])
  @@index([healthUnitTaxId, specialityId])
}
```

**Regra de negócio:** só aceitar `ProfessionalWorkTime` se existir `ProfessionalAssignment` activo para a mesma tripla (`professionalTaxId`, `healthUnitTaxId`, `specialityId`).

---

### 2.2. Evitar **overlaps** de WorkTime

Para impedir turnos sobrepostos do **mesmo** profissional/unidade/especialidade/weekday:

1) **Validação na aplicação** (detecção de overlap).
2) **Constraint no Postgres** (robusto) com `tstzrange` + `EXCLUDE USING GIST`.

**Migração SQL (exemplo):**
```sql
-- Coluna gerada com o range do turno [startAt, endsAt)
ALTER TABLE "ProfessionalWorkTime"
ADD COLUMN "time_range" tstzrange
  GENERATED ALWAYS AS (tstzrange("startAt","endsAt",'[)')) STORED;

-- Necessário para GIST com chaves mistas
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- Evita overlap para a MESMA tripla + weekday, quando isActive=true
ALTER TABLE "ProfessionalWorkTime"
  ADD CONSTRAINT professional_worktime_no_overlap
  EXCLUDE USING GIST (
    "professionalTaxId" WITH =,
    "healthUnitTaxId"   WITH =,
    "specialityId"      WITH =,
    "weekDay"           WITH =,
    "time_range"        WITH &&
  )
  WHERE ("isActive" = true);
```

> **Turnos que cruzam meia-noite**: ver §5.

---

## 3) “Contrato” entre Agenda e Disponibilidade

- `Schedules.healthUnitTaxId` e `Schedules.specialityId` **filtros base** para WorkTime.
- `Schedules.typeOfService` filtra WorkTime com `typeOfService` **igual** ou `null` (null = “serve todos”).
- `eachSlotInheritAllScheduleProfessionals` (ou `inheritAllProfessionals` nos Slots)  
  → quando **true**, a lista efectiva do Slot é a **interseção**:
  ```
  availableProfessionalTaxIds (da Agenda)
  ∩
  Profissionais com WorkTime válido no dia/hora
  ```
- Quando **false**, podes usar Slots específicos por profissional — mas **ainda assim** exigir WorkTime compatível.

---

## 4) Regras de Alocação

### 4.1. Agenda por **duração** (hours grid)
A **timebox `h`** só é válida para um profissional se:
- `h` ∈ `[WorkTime.startAt, WorkTime.endsAt)` **no mesmo** `weekDay`;
- Unidade & especialidade batem;
- `typeOfService` do Schedule é compatível com o WorkTime;
- `validFrom/validTo` cobrem o dia/hora.

### 4.2. Agenda por **capacidade** (sem `hours`)
A marcação sem hora (apenas por `slotId`) só é permitida se a **interseção de tempo do dia** entre Agenda e WorkTime for **não vazia**.

---

## 5) Turnos que cruzam a meia-noite

Escolhe **uma abordagem** e sê consistente:

**A1) Normalizar em input (recomendado):**  
dividir o turno em **duas linhas**:
- seg 20:00–23:59:59
- ter 00:00–02:00

**A2) Calcular em runtime:**  
se `endsAt < startAt (no mesmo dia lógico)`, dividir em:
- `[startAt, fim_do_dia]` no `weekDay` X
- `[início_do_dia, endsAt]` no `weekDay` X+1 (mod 7)

> A1 facilita índices e constraints; A2 evita duplicar registos, mas complica o cálculo.

---

## 6) Pseudocódigo de Interseção (por duração)

```ts
function canProfessionalTakeHour({
  professionalTaxId, healthUnitTaxId, specialityId, typeOfService,
  scheduleWindowUtc, // [start,end) do dia
  hourUtc,           // timebox concreta
}): boolean {
  const wday = hourUtc.getUTCDay(); // 0..6

  const worktimes = repo.findWorkTimes({
    professionalTaxId,
    healthUnitTaxId,
    specialityId,
    weekDay: wday,
    // aceitar também null no turno
    typeOfService,
    validOn: hourUtc, // dentro de [validFrom, validTo], se definidos
    isActive: true
  });

  return worktimes.some(wt =>
    hourUtc >= wt.startAt && hourUtc < wt.endsAt
    // se usares A2 (cross-midnight runtime), aqui fazes a lógica de split
  );
}
```

**Por capacidade:** substituir `hourUtc` por verificação de **interseção não vazia** entre `scheduleWindowUtc` e o WorkTime do dia.

---

## 7) Fluxo Operacional

1) **Sync de sombras** (`ProfessionalShadow`, `SpecialityShadow`) do micro de referência.  
2) **Criar/actualizar** `ProfessionalAssignment` (profissional ↔ unidade ↔ especialidade).  
3) **Validar e gravar** `ProfessionalWorkTime` **apenas se** existir `ProfessionalAssignment` activo e **sem** overlap.  
4) **Gerar Slots**: para cada `dateUtc`:
   - expandir recorrência da Agenda;
   - filtrar profissionais elegíveis via `availableProfessionalTaxIds` ∩ WorkTime válido;
   - **por duração**: gerar timeboxes e marcar indisponíveis fora do WorkTime;  
     **por capacidade**: permitir marcação no dia apenas se a interseção existir.
5) **Marcação** (Appointment):
   - por duração: `selectedHour` deve estar **AVAILABLE** e dentro do WorkTime;
   - por capacidade: não exceder `markingLimit` e garantir interseção do dia.

---

## 8) Índices & Constraints (resumo)

- `ProfessionalAssignment`  
  `@@unique([professionalTaxId, healthUnitTaxId, specialityId])`  
  `@@index([healthUnitTaxId, specialityId])`

- `ProfessionalWorkTime`  
  `@@index([professionalTaxId, healthUnitTaxId, weekDay])`  
  `@@index([professionalTaxId, healthUnitTaxId, specialityId])`  
  `@@index([isActive, weekDay])`  
  `@@index([validFrom, validTo])`  
  **Postgres EXCLUDE** para **não sobrepor** turnos (ver §2.2).

---

## 9) Regras de Negócio (Checklist)

1. **Obrigatório** existir `ProfessionalAssignment` activo para criar `ProfessionalWorkTime`.  
2. **Proibido overlap** de WorkTime para a mesma tripla + weekday.  
3. **Interseção necessária** entre Agenda e WorkTime para alocar profissional no dia/hora.  
4. **Por duração**: cada timebox é válida só se estiver no range do WorkTime.  
5. **Por capacidade**: só marca se a interseção (dia) for > 0.  
6. **Cross-midnight**: escolher A1 (input em duas linhas) **ou** A2 (cálculo em runtime).  
7. **Sombras** são **read-only**; `externalHash`/`fetchedAt` ajudam em idempotência e auditoria.  
8. **TypeOfService** do Schedule deve ser compatível com o WorkTime (igual ou WorkTime null).

---

## 10) Impacto em Performance & Robustez

- **Vantagens** de `externalHash`/`sourceSystem` nos shadows:
  - Idempotência de sync (evita re-writes desnecessários).
  - Auditoria e diagnóstico (qual payload externo gerou o estado actual).
- **Overhead** marginal na escrita; **ganho** em estabilidade e previsibilidade.  
- **Caching** recomendado na leitura (e.g., WorkTimes activos por unidade/especialidade/weekday).

---

## 11) Exemplos de Consultas (IDEIA)

**Encontrar WorkTimes válidos para um dia/hora (por duração):**
```sql
-- parâmetros: professionalTaxId, healthUnitTaxId, specialityId, hourUtc
SELECT *
FROM "ProfessionalWorkTime" wt
WHERE wt."professionalTaxId" = $1
  AND wt."healthUnitTaxId"   = $2
  AND wt."specialityId"      = $3
  AND wt."weekDay"           = extract(dow from $4)::int  -- 0..6
  AND wt."isActive"          = true
  AND (wt."typeOfService" IS NULL OR wt."typeOfService" = $5)
  AND (wt."validFrom" IS NULL OR wt."validFrom" <= $4)
  AND (wt."validTo"   IS NULL OR wt."validTo"   >= $4)
  AND $4 >= wt."startAt" AND $4 < wt."endsAt";
```

**Por capacidade (sem hora), verificar interseção com a janela do Schedule do dia:**
```sql
-- parâmetros: dateStartUtc, dateEndUtc (janela do dia), + filtros acima
SELECT *
FROM "ProfessionalWorkTime" wt
WHERE ...
  AND tstzrange(wt."startAt", wt."endsAt",'[)') && tstzrange($1, $2,'[)');
```

---

### Conclusão

Com **ProfessionalAssignment** (tripla profissional/unidade/especialidade), **ProfessionalWorkTime** sem overlaps, e as **regras de interseção** na geração de Slots e na marcação, o sistema respeita a disponibilidade real do profissional em **cada hospital**, **cada especialidade**, e **cada dia/horário**, suportando tanto agendas **por duração** quanto **por capacidade** — de forma robusta e auditável.
