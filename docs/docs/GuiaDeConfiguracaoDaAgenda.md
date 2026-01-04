# **Contexto: Criação/Actualização de Agenda**

> Todas as datas em **UTC** (ISO 8601). Padrões: `page=1`, `limit=20`, `sortOrder=desc`.  
> Autenticação: Bearer Token. Autorização: RBAC/ABAC conforme perfis.

Este guia descreve tudo o que deve ser observado ao **criar** ou **actualizar** uma Agenda (`Schedules`). Inclui **pré-condições**, **validações**, **regras de negócio**, **padrões de concorrência**, **contratos de API**, **exemplos de payload**, **erros**, **auditoria**, **segurança**, **testes** e **boas práticas**.

---

## Sumário
- [A. Objectivo do processo](#a-objectivo-do-processo)
- [B. Pré-condições funcionais](#b-pre-condicoes-funcionais)
- [C. Mapeamento ao modelo (resumo)](#c-mapeamento-ao-modelo-resumo)
- [D. Validações (camada de negócio)](#d-validacoes-camada-de-negocio)
- [E. Regras de negócio (passo a passo)](#e-regras-de-negocio-passo-a-passo)
- [F. Cálculos derivados e políticas](#f-calculos-derivados-e-politicas)
- [G. Contratos de API (sugestão)](#g-contratos-de-api)
- [H. Concorrência, transacções e integridade](#h-concorrencia-transaccoes-e-integridade)
- [I. Auditoria e soft-delete](#i-auditoria-e-soft-delete)
- [J. Segurança, permissões e limites](#j-seguranca-permissoes-e-limites)
- [K. Erros padronizados (sugestão)](#k-erros-padronizados)
- [L. Exemplos adicionais (válidos e inválidos)](#l-exemplos-adicionais-validos-e-invalidos)
- [M. Pseudo-código do service (Create/Update)](#m-pseudo-codigo-do-service-createupdate)
- [N. Testes (checklist)](#n-testes)
- [O. Boas práticas e dicas finais](#o-boas-praticas-e-dicas-finais)

---

## A. Objectivo do processo

Criar/actualizar uma entidade **Schedules** que define:

- Janela diária de atendimento (**início/fim**).
- Recorrência (**NONE/DAILY/WEEKLY/…/CUSTOM**).
- **Dias da semana** activos.
- **Vigência** (período de validade).
- **Profissionais** elegíveis.
- Políticas de **marcação/cancelamento**.
- **Geração automática** de slots (opcional).
- **Exclusões** (dias/intervalos) associadas a outras entidades.
- **Compatibilidade total** com fluxos de **Slots**, **Exclusões**, **Remarcações**, **Encaminhamentos** e **Relatórios** já definidos.

---

## B. Pré-condições funcionais

### 1) Contexto clínico definido
- `healthUnitTaxId` (unidade) **válido** no ecossistema.
- `specialityId` **válido** no catálogo de especialidades.

### 2) Tempo e fuso horário
- Backend e BD a trabalhar em **UTC** (`timestamptz`).
- Frontend converte para o **fuso do utilizador** apenas na apresentação.

### 3) Parâmetros mínimos

> Conjunto mínimo de campos e regras para **criar/actualizar** uma agenda válida.

- **`typeOfService`** *(obrigatório)*  
  Valores: `IN_PERSON` | `TELEMEDICINE` | `HOME`  
  _Define o modo de atendimento (impacta UI e validações como `roomLink` para telemedicina)._

- **`typeOfSchedule`** *(obrigatório)*  
  Valores: `CONSULTATION` | `VACCINE` | `EXAM` | `RETURN`  
  _Define o tipo de acto clínico (pode activar políticas específicas)._

- **Janela diária** *(obrigatória)*  
  - `startTime` **<** `endTime` (ambos em `timestamptz`/UTC).  
  - _Delimita o período diário onde se geram horas/slots._

- **Vigência** *(obrigatória em `startDate`; `endDate` opcional)*  
  - `startDate` (UTC) **≤** `endDate` (se informado).  
  - _Define desde quando a agenda é válida e, opcionalmente, até quando._

- **Recorrência & dias activos**  
  - `typeOfRecurrence` ≠ `NONE` ⇒ `weekDays.length ≥ 1`.  
  - _Sem dias activos, não há repetição semanal/mensal._

- **Ritmo de marcação (XOR — escolher uma via)**  
  - **Via A (por duração):** `appointmentDurationInMinutes` > 0  
    _Divide a janela diária em blocos fixos (ex.: 20/30 min)._  
  - **Via B (por capacidade):** `maxAppointmentsPerSlot` ≥ 1  
    _“Dia como slot” ou overbooking controlado por limite._

> **Fail-fast (validações recomendadas):**  
> - Rejeitar se faltar `typeOfService` ou `typeOfSchedule`.  
> - Rejeitar se `startTime >= endTime`.  
> - Rejeitar se `typeOfRecurrence ≠ NONE` **e** `weekDays` estiver vazio.  
> - Rejeitar se **nem** `appointmentDurationInMinutes` **nem** `maxAppointmentsPerSlot` forem fornecidos no mesmo pedido.

### 4) Capacidade/tempo

- **XOR obrigatório (escolher uma via):**
  - **Via A — por duração:** `appointmentDurationInMinutes` > 0  
    - Ex.: 30 → gera 08:00, 08:30, 09:00, … até `endTime`.  
    - **Cálculo:**  
      - `janela_util_min = endTime - startTime` (em minutos)  
      - `num_slots = floor(janela_util_min / appointmentDurationInMinutes)`  

  - **Via B — por capacidade:** `maxAppointmentsPerSlot` ≥ 1  
    - Útil para “**dia como slot**” (sem grade por minutos) ou **overbooking** controlado.  
    - Ex.: `maxAppointmentsPerSlot = 20` → até 20 marcações distribuídas no dia.

- **Derivação (quando ambos vierem):**  
  - Se os dois forem enviados, **duração** governa a geração de horários;  
  - `maxAppointmentsPerSlot` funciona como **limite por horário** (multi-atendimento).

- **Validações obrigatórias:**  
  - `startTime < endTime` (sempre).  
  - `appointmentDurationInMinutes > 0` (quando usado).  
  - `maxAppointmentsPerSlot ≥ 1` (quando usado).  
  - **Pelo menos um** entre duração e capacidade.

#### Erros possíveis e códigos recomendados

| Código | Quando | Exemplo de `errors[]` |
|-------:|-------|------------------------|
| **400** | Payload mal formado (tipos errados), campos desconhecidos, formato inválido. | `{"field":"appointmentDurationInMinutes","message":"Must be an integer"}` |
| **401** | Sem token ou sessão inválida. | `{"message":"Unauthorized"}` |
| **403** | Actor sem permissões para gerir a agenda da unidade. | `{"message":"Forbidden for healthUnitTaxId=540102938"}` |
| **404** | Actualização de agenda inexistente. | `{"message":"Schedule not found"}` |
| **409** | Regras mutuamente exclusivas violadas ou duplicidade lógica. | `{"message":"Provide either appointmentDurationInMinutes OR maxAppointmentsPerSlot, not both null"}` |
| **409** | Janela impossível para a duração (ex.: 50 min numa janela de 45 min quando política não permite o último slot parcial). | `{"message":"Duration does not fit within startTime/endTime under current policy"}` |
| **409** | A alteração colide com **slots/consultas já existentes** (política não-permissiva). | `{"message":"Existing appointments conflict with the new duration window"}` |
| **422** | Regras de negócio falhadas (payload sintáctico ok, mas inválido semântico). | `{"field":"maxAppointmentsPerSlot","message":"Must be >= 1"}` |
| **422** | Nenhuma via escolhida (duração **e** capacidade ausentes). | `{"message":"Provide appointmentDurationInMinutes or maxAppointmentsPerSlot"}` |
| **422** | `startTime >= endTime`. | `{"message":"startTime must be before endTime"}` |
| **429** | Protecção contra abuso/automação (criações em massa). | `{"message":"Rate limit exceeded. Try later."}` |
| **500** | Erro inesperado do servidor. | `{"message":"Internal error","correlationId":"..."} ` |

**Exemplo (422 — XOR não cumprido)**  
```json
{
  "message": "Provide appointmentDurationInMinutes or maxAppointmentsPerSlot",
  "errors": [
    {"field":"appointmentDurationInMinutes","message":"Missing or invalid"},
    {"field":"maxAppointmentsPerSlot","message":"Missing or invalid"}
  ]
}
```

---

### 5) Recorrência e dias

> Define **quando** a agenda se repete (frequência) e **em quais dias da semana** é válida.  
> **Regra principal:** se `typeOfRecurrence` **≠** `NONE`, então `weekDays` **deve conter pelo menos 1 dia**.

#### 5.1 Objectivo
- Determinar **em que dias** a janela diária (`startTime` → `endTime`) se aplica ao longo da **vigência** (`startDate` → `endDate`).
- Controlar **padrões de repetição** (semanal, mensal, etc.) com base em `typeOfRecurrence` e, opcionalmente, regras avançadas (RRULE noutros pontos do sistema).

#### 5.2 Comportamento por tipo de recorrência
- `NONE`: sem repetição (slots apenas manuais). `weekDays` pode ser vazio.
- `DAILY`: todos os dias na vigência; `weekDays` opcional para restringir (ex.: dias úteis).
- `WEEKLY`: repete nos `weekDays` (obrigatório ≥ 1).
- `MONTHLY/QUARTERLY/SEMIANNUALLY/ANNUALLY`: repetição por períodos; pode delegar para **RRULE**.
- `CUSTOM`: padrões personalizados; preferir **RRULE**. Se não houver RRULE, aplicar validações equivalentes.

#### 5.3 Regras e validações obrigatórias
- `typeOfRecurrence ≠ NONE` ⇒ `weekDays.length ≥ 1`.
- `weekDays` ∈ `SUNDAY..SATURDAY`, **sem duplicados**.
- `startDate ≤ endDate` (quando `endDate` existir).  
- **Exclusões** têm prioridade sobre recorrência (`ExcludeDay` > `ExcludeRange`).

#### 5.4 Exemplos e erros
**400** — `weekDays` inválido/duplicado  
```json
{
  "message": "Invalid weekDays payload",
  "errors": [
    { "field": "weekDays[0]", "message": "Invalid enum value 'MONDEY'. Use SUNDAY..SATURDAY" },
    { "field": "weekDays", "message": "Duplicate values are not allowed" }
  ]
}
```

---

## C. Mapeamento ao modelo

```prisma
model Schedules {
  id                            String                    @id @default(uuid())
  healthUnitTaxId               String 
  specialityId                  String 
  weekDays                      WeekDays[]                @default([])
  startTime                     DateTime                  @db.Timestamptz 
  endTime                       DateTime                  @db.Timestamptz 
  appointmentDurationInMinutes  Int? 
  maxAppointmentsPerSlot        Int?
  typeOfRecurrence              TypeOfRecurrence 
  excludeRanges                 Schedules_ExcludeRange_[] @relation("Schedule") 
  excludeDays                   Schedule_ExcludeDay_[]    @relation("Schedule") 
  startDate                     DateTime                  @db.Timestamptz 
  endDate                       DateTime?                 @db.Timestamptz 
  typeOfSchedule                TypeOfSchedule 
  typeOfService                 TypeOfService 
  advanceCancellationInHours    Int? 
  deadlineForSlotBookingInHours Int? 
  availableProfessionalTaxIds   String[] 
  automaticallyGenerateSlots    Boolean 
  eachSlotInheritAllScheduleProfessionals Boolean @default(false)
  applyGenderRestrictions GenderRestrictions @default(NONE)
  autoPublishSlots          Boolean @default(false) 
  requireSupervisorApproval Boolean @default(true)
  defaultPublishBufferHours Int? 
  defaultReviewTTLHours     Int?
  createdBy    String 
  slots        Slots[]
  appointments Appointment[]
  createdAt    DateTime      @default(now()) @db.Timestamptz
  updatedAt    DateTime      @updatedAt @db.Timestamptz
  deletedAt    DateTime?

  @@index([healthUnitTaxId, specialityId])
  @@index([createdAt])
  @@index([deletedAt])
}
```

---

## D. Validações (camada de negócio)

> **Ordem recomendada:** formatos → coerência temporal → recorrência/dias → capacidade → políticas → segurança.

1) **Formatos**  
- `healthUnitTaxId`, `specialityId`, `createdBy`: **strings não vazias**.  
- Listas (`weekDays`, `availableProfessionalTaxIds`) podem ser **vazias**, excepto quando regras obriguem.  
- Datas: **ISO 8601**, guardadas como **timestamptz (UTC)**.

2) **Coerência temporal**  
- `startTime` **<** `endTime`.  
- `endDate` (se existir): `startDate` **≤** `endDate`.  
- `NONE` aceita `endDate` nulo.

3) **Recorrência & dias**  
- `NONE` ⇒ `weekDays` pode ser vazio.  
- `≠ NONE` ⇒ `weekDays` obrigatório (≥ 1, sem duplicados, enum válido).

4) **Capacidade/Tempo (XOR)**  
- **A)** `appointmentDurationInMinutes` > 0 **OU**  
- **B)** `maxAppointmentsPerSlot` ≥ 1.  
- Ambos presentes ⇒ duração governa; capacidade vira limite por hora (multi-atendimento).

5) **Políticas**  
- `deadlineForSlotBookingInHours` ≥ 0; `advanceCancellationInHours` ≥ 0  
- (Recomendado) `advanceCancellationInHours` ≥ `deadlineForSlotBookingInHours`.

6) **Profissionais**  
- `availableProfessionalTaxIds`: lista de IDs externos (validar via micro-serviço de utilizadores ou cache).

7) **Segurança/ACL**  
- `createdBy` deve ter **permissão** para gerir agendas nessa unidade.

---

## E. Regras de negócio (passo a passo)

### 1) Receber pedido (Create/Update) com payload JSON
- Validar `Content-Type: application/json`.
- Possíveis erros: `400/401/403/405/409/415` (detalhados acima).

### 2) Normalizar datas/horas para UTC
- Converter tudo para UTC (armazenar em `timestamptz`).
- Erros: `400` (formato), `422` (coerência).

### 3) Validar (secção D) e retornar erros agregados
- `400` (formato), `403` (ACL), `422` (regras).
- **Boas práticas:** listar **todos** os erros em `errors[]`.

### 4) Coerência operacional — Tabela de erros
(ver tabela já incluída na tua versão consolidada)

### 5) Persistir a Agenda (transacção) — Tabela de erros
(ver tabela já incluída na tua versão consolidada)

### 6) Geração automática de slots (`automaticallyGenerateSlots = true`)
- **Objectivo**: criar Slots/horas conforme a Agenda + Exclusões.
- **Estratégia**: preferir **assíncrono** com `operationId` (ver `/operations/{id}`).
- **Idempotência**: chave `(scheduleId, from, to)`.
- **Aplicação de exclusões**: `ExcludeDay` > `ExcludeRange`.
- **Upsert** de Slots por `(scheduleId, specificDate)` sem apagar histórico.

**Pseudo-código do worker (resumo)**
```ts
async function generateSlotsJob({ scheduleId, from, to, correlationId }) {
  // 1) Validar, garantir idempotência e expandir recorrência
  // 2) Aplicar exclusões (dia > intervalo)
  // 3) Construir grelha (via duração ou via capacidade/dia)
  // 4) Upsert idempotente de Slots e publicar métricas/eventos
}
```

**Tabelas de erros por fase**: incluídas na versão consolidada (A..F).

### 7) Publicar eventos (opcional)
- `schedule.created`, `schedule.updated`, `schedule.slots.generated`, `schedule.slots.generation.failed`.

---

## F. Cálculos derivados e políticas

1) **Capacidade derivada** (quando há duração)  
- `slots_por_dia = floor((end - start) / appointmentDurationInMinutes)`  
- `maxAppointmentsPerSlot` pode permitir **multi-atendimento** por hora.

2) **Validade da agenda**  
- `NONE`: sem geração por recorrência; aceita **slots manuais**.  
- `WEEKLY/MONTHLY/...`: gera automaticamente conforme janela.

3) **Compatibilidade com exclusões**  
- Exclusões consideradas na **geração** e **em tempo real** na marcação.

---

## G. Contratos de API

### 1) Criar Agenda — `POST /schedules`
**Request (exemplo):**
```json
{
  "healthUnitTaxId": "540102938",
  "specialityId": "CARDIOLOGY",
  "weekDays": ["MONDAY", "WEDNESDAY", "FRIDAY"],
  "startTime": "2025-11-01T08:00:00Z",
  "endTime": "2025-11-01T16:00:00Z",
  "appointmentDurationInMinutes": 30,
  "maxAppointmentsPerSlot": null,
  "typeOfRecurrence": "WEEKLY",
  "startDate": "2025-11-01T00:00:00Z",
  "endDate": "2026-01-31T00:00:00Z",
  "typeOfSchedule": "CONSULTATION",
  "typeOfService": "TELEMEDICINE",
  "advanceCancellationInHours": 12,
  "deadlineForSlotBookingInHours": 2,
  "requireSupervisorApproval": true,
  "autoPublishSlots": false,
  "defaultPublishBufferHours": 4,
  "defaultReviewTTLHours": 24,
  "eachSlotInheritAllScheduleProfessionals": true,
  "availableProfessionalTaxIds": ["BI12345", "BI67890"],
  "automaticallyGenerateSlots": true,
  "createdBy": "admin-user"
}
```

**Responses:**
- **201 Created** — corpo com recurso e `id` (+ `ETag`).
- **400/409/415/422** — conforme tabelas.
- **Cabeçalhos**: `ETag`, `Correlation-Id`.

### 2) Actualizar Agenda — `PUT/PATCH /schedules/{id}`
- **PATCH** aceita **campos parciais**.
- Recomendar `If-Match` (ETag) para **concorrência optimista**.

**Cuidados especiais no Update**
- Alterações estruturais (janela/duração/recorrência) **não** reescrevem slots passados.
- Usar **`effectiveFrom`** para cortes futuros e endpoint dedicado para **regenerar**.

---

## H. Concorrência, transacções e integridade

- Transacção ao persistir `Schedules`.
- Geração de slots numa **transacção separada** (ou job assíncrono).
- Índices: já listados no modelo.
- **Locks lógicos** para evitar jobs concorrentes na mesma janela.

**Invariantes**
- `startTime < endTime`, `startDate ≤ endDate`.
- `typeOfRecurrence ≠ NONE` ⇒ `weekDays` não vazio.
- Pelo menos um entre **duração** e **capacidade**.

---

## I. Auditoria e soft-delete

- `createdAt`, `updatedAt` automáticos.
- **Soft-delete**: `deletedAt`.
- Listagens excluem `deletedAt != NULL` por omissão.
- **Audit log** (opcional): diffs antes/depois.

---

## J. Segurança, permissões e limites

- **ACL** por `healthUnitTaxId` / papel do actor.
- Validação cruzada de profissionais via micro-serviço (com **cache**).
- **Rate limit** para evitar alterações em massa.

---

## K. Erros padronizados

| Código | Quando                 | Corpo (exemplo)                                                                 |
|-------:|------------------------|----------------------------------------------------------------------------------|
| 400    | Validação falhou       | `{"errors":[{"field":"startTime","message":"startTime must be before endTime"}]}` |
| 401    | Não autenticado        | `{"message":"Unauthorized"}`                                                    |
| 403    | Sem permissão          | `{"message":"Forbidden"}`                                                       |
| 404    | Agenda não encontrada  | `{"message":"Schedule not found"}`                                              |
| 409    | Conflito lógico        | `{"message":"A schedule with same unit/speciality and window already exists"}`  |
| 500    | Erro interno           | `{"message":"Internal server error","correlationId":"..."}`                     |

---

## L. Exemplos adicionais (válidos e inválidos)

### ✅ Válido — Agenda “dia como slot” (sem duração)
```json
{
  "healthUnitTaxId": "5002159961",
  "specialityId": "DERMATOLOGY",
  "weekDays": ["TUESDAY", "THURSDAY"],
  "startTime": "2025-11-01T00:00:00Z",
  "endTime": "2025-11-01T23:59:59Z",
  "maxAppointmentsPerSlot": 20,
  "typeOfRecurrence": "WEEKLY",
  "startDate": "2025-11-01T00:00:00Z",
  "typeOfSchedule": "CONSULTATION",
  "typeOfService": "IN_PERSON",
  "automaticallyGenerateSlots": true,
  "createdBy": "ops"
}
```

### ❌ Inválido — Recorrência sem dias
```json
{
  "healthUnitTaxId": "5002159961",
  "specialityId": "CARDIOLOGY",
  "weekDays": [],
  "startTime": "2025-11-01T08:00:00Z",
  "endTime": "2025-11-01T16:00:00Z",
  "appointmentDurationInMinutes": 30,
  "typeOfRecurrence": "WEEKLY",
  "startDate": "2025-11-01T00:00:00Z",
  "typeOfSchedule": "CONSULTATION",
  "typeOfService": "IN_PERSON",
  "automaticallyGenerateSlots": true,
  "createdBy": "ops"
}
```
**Erro esperado:** `400 — "weekDays não pode ser vazio quando typeOfRecurrence ≠ NONE"`.

### ❌ Inválido — Nem duração nem capacidade
```json
{
  "healthUnitTaxId": "5002159961",
  "specialityId": "CARDIOLOGY",
  "weekDays": ["MONDAY"],
  "startTime": "2025-11-01T08:00:00Z",
  "endTime": "2025-11-01T16:00:00Z",
  "typeOfRecurrence": "WEEKLY",
  "startDate": "2025-11-01T00:00:00Z",
  "typeOfSchedule": "CONSULTATION",
  "typeOfService": "IN_PERSON",
  "automaticallyGenerateSlots": true,
  "createdBy": "ops"
}
```
**Erro esperado:** `400 — "Forneça appointmentDurationInMinutes ou maxAppointmentsPerSlot"`.

---

## M. Pseudo-código do service (Create/Update)

```ts
async function upsertSchedule(input, actor) {
  normalizeToUTC(input); // startTime, endTime, startDate, endDate

  // 1) Validações
  ensureNotEmpty(input.healthUnitTaxId, "healthUnitTaxId");
  ensureNotEmpty(input.specialityId, "specialityId");
  assert(input.startTime < input.endTime, "startTime must be before endTime");
  if (input.endDate) assert(input.startDate <= input.endDate, "startDate <= endDate");

  const hasDuration = isPositiveInt(input.appointmentDurationInMinutes);
  const hasCapacity = isPositiveInt(input.maxAppointmentsPerSlot);
  assert(hasDuration || hasCapacity, "duration OR capacity required");

  if (input.typeOfRecurrence !== "NONE") {
    assert(input.weekDays && input.weekDays.length > 0, "weekDays required for recurrence");
  }

  checkPermissions(actor, input.healthUnitTaxId); // ACL

  // 2) Persistência (transação)
  return db.$transaction(async (tx) => {
    const schedule = await tx.schedules.upsert({
      where: { id: input.id ?? "__none__" },
      create: {
        ...input,
        availableProfessionalTaxIds: input.availableProfessionalTaxIds ?? [],
        createdBy: actor.id
      },
      update: { ...input }
    });

    // 3) Geração de slots (opcional)
    if (input.automaticallyGenerateSlots) {
      await enqueueSlotGenerationJob(schedule.id, {
        from: input.startDate,
        to: input.endDate ?? addMonths(input.startDate, 2) // janela operacional
      });
    }

    // 4) Evento (opcional)
    await publish("schedule.updated", { id: schedule.id });

    return schedule;
  });
}
```

---

## N. Testes

- Criação válida com **duração** (30 min).
- Criação válida **“dia como slot”** (sem duração, com capacidade).
- Actualização que altera apenas **políticas** (deadlines) — **não** deve mexer em slots **históricos**.
- Actualização de `weekDays` com `typeOfRecurrence ≠ NONE` — **válido**.
- Erro: `startTime >= endTime`.
- Erro: `typeOfRecurrence ≠ NONE` e `weekDays = []`.
- Erro: **sem** duração e **sem** capacidade.
- Timezone: guardar sempre **timestamptz (UTC)**.
- Permissões: utilizador **sem ACL** não pode criar/alterar.
- Geração automática: ao activar, **job** é agendado (ou executado) e **slots** são gerados.
- Compatibilidade: criação/actualização **não deve quebrar** exclusões existentes.
- Soft-delete: agendas marcadas em `deletedAt` não surgem por omissão.

---

## O. Boas práticas e dicas finais

- Separar **“definição”** da agenda da **geração de slots** (job/serviço dedicado).
- **Não** reescrever slots **passados** ao actualizar a agenda (evita perda de histórico).
- Mensagens claras ao cliente: “A agenda foi criada. **Os horários serão gerados automaticamente**.”
- Documentar políticas locais: **tempos de tolerância**, **feriados padrão**, **janelas de geração**.
- Observabilidade: logs com `scheduleId`, `healthUnitTaxId`, `specialityId` e `correlationId`.

---

> **Compatibilidade**: Este guia já está alinhado com os módulos de **Slots**, **Exclusões**, **Relatórios**, **Remarcações** e **Encaminhamentos**, incluindo as políticas de idempotência, concorrência (ETag/If-Match) e geração assíncrona com `/operations/{id}`.
