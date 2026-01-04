# Modelos de Exclusão – Baika + Saúde (Atualizado pelo novo schema)

Este documento descreve como configurar **bloqueios de disponibilidade** usando os modelos `ExcludeRange` e `ExcludeDay`, de acordo com o **schema atualizado**.

> **Prioridade geral:** `ExcludeDay` (bloqueia o **dia inteiro**) **>** `ExcludeRange` (bloqueia **janelas/partes do dia**).  
> **Timezone:** Todas as datas/horas devem ser armazenadas e processadas em **UTC (timestamptz)**.

---

## Modelo `ExcludeRange` (bloqueios parciais / janelas)

Usado para **bloquear intervalos de tempo dentro de um dia** (ex.: almoço 12:00–13:00), por **dias específicos** ou **recorrências** (diária, semanal, RRULE), com **escopo** por **unidade** e **agendas**.

### Estrutura (conforme schema)

| Campo | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| `id` | `String @id @default(uuid())` | Sim | Identificador único. |
| `startTime` | `DateTime?` | Não | **Hora** inicial para bloqueios **recorrentes** (interpretação “relógio” em UTC). |
| `endTime` | `DateTime?` | Não | **Hora** final do bloqueio recorrente. |
| `startDate` | `DateTime? @db.Timestamptz` | Não | **Data/hora** inicial para bloqueios **pontuais ou de vários dias**. |
| `endDate` | `DateTime? @db.Timestamptz` | Não | **Data/hora** final do bloqueio pontual. |
| `excludeForAllSlots` | `Boolean @default(false)` | Sim | Se `true`, remove **todos os horários** do(s) slot(s) na janela; se `false`, permite granularidade por lista/dia. |
| `excludeFor` | `WeekDays[] @default([])` | Não | **Dias da semana** aplicáveis quando `typeOfRecurrence ≠ NONE` (ignorado se `excludeForAllSlots = true`). |
| `excludeForSpecificDates` | `DateTime[] @default([])` | Não | Datas **pontuais** a bloquear (p. ex., dias soltos no mês). |
| `typeOfRecurrence` | `TypeOfRecurrence @default(NONE)` | Sim | Frequência: `NONE/DAILY/WEEKLY/MONTHLY/.../CUSTOM`. |
| `isActive` | `Boolean @default(true)` | Sim | Liga/desliga o bloqueio. |
| `healthUnitTaxId` | `String` | Sim | Escopo: unidade de saúde a que o bloqueio pertence. |
| `includeForAllUnitSchedules` | `Boolean @default(false)` | Sim | Se `true`, aplica-se a **todas** as agendas da unidade (`healthUnitTaxId`). |
| `assignedSchedules` | `Schedules_ExcludeRange_[] @relation("Range")` | Condicional | Relação M:N para **vincular agendas específicas** quando `includeForAllUnitSchedules = false`. |
| `definedBy` | `String` | Sim | Identificador do utilizador/sistema que criou. |
| `rrule` | `String?` | Não | Regra iCalendar (RFC 5545), p. ex. `FREQ=WEEKLY;INTERVAL=2;BYDAY=SA`. |
| `title` | `String` | Sim | Título amigável (“Intervalo de almoço”, “Formação interna”). |
| `reason` | `String?` | Não | Motivo/descrição. |
| `deletedAt` | `DateTime?` | Não | Soft-delete. |
| `deletedBy` | `String?` | Não | Quem eliminou. |
| `updatedAt` | `DateTime @updatedAt` | Sim | Auditoria. |
| `updatedBy` | `String` | Sim | Quem actualizou por último. |

#### Invariantes de Governação (escopo)
- **A)** Se `includeForAllUnitSchedules = true` ⇒ **`assignedSchedules` deve estar vazio** (aplica-se a todas as agendas da unidade).  
- **B)** Se `includeForAllUnitSchedules = false` ⇒ **`assignedSchedules` deve ter pelo menos 1 agenda**.  
> Caso contrário, o escopo fica ambíguo.

#### Validações Essenciais
- Quando usar `startTime`/`endTime` (recorrente), **ambos** devem existir e `startTime < endTime`.  
- Em bloqueios pontuais, `startDate <= endDate` (quando ambos informados).  
- `typeOfRecurrence ≠ NONE` e **sem `excludeFor`/`rrule`/`excludeForSpecificDates`** ⇒ comportamento deve ser **claramente definido** (recomenda-se exigir ao menos uma dessas âncoras).  
- `isActive = false` ⇒ o bloqueio **não deve** ser aplicado na geração/cálculo.

#### Interpretação (engine)
1. Determina **escopo**: unidade (`healthUnitTaxId`) e **conjunto de agendas** (`includeForAllUnitSchedules` vs `assignedSchedules`).  
2. Expande **recorrências** (`typeOfRecurrence` ou `rrule`) e/ou **datas específicas** (`excludeForSpecificDates`).  
3. Para cada **dia alvo**:
   - Se dia foi totalmente excluído por **ExcludeDay** ⇒ **ignorar ExcludeRange** (prioridade do dia).  
   - Caso contrário, aplique a **janelagem** `startTime → endTime` e remova/etiquete horas como `BLOCKED`.  
4. Nunca remover horas com `status ∈ {BOOKED, HELD}`; no máximo, **marcar como `BLOCKED`** para futuras marcações.

#### Exemplos (payloads lógicos)
- **Intervalo diário (todos os schedules da unidade):**
```json
{
  "title": "Almoço",
  "reason": "Pausa operacional",
  "typeOfRecurrence": "DAILY",
  "startTime": "1970-01-01T12:00:00Z",
  "endTime": "1970-01-01T13:00:00Z",
  "excludeForAllSlots": true,
  "includeForAllUnitSchedules": true,
  "healthUnitTaxId": "5002159961",
  "definedBy": "ops@baika"
}
```
- **Formação semanal (apenas QUARTAS de agendas específicas):**
```json
{
  "title": "Formação Interna",
  "typeOfRecurrence": "WEEKLY",
  "excludeFor": ["WEDNESDAY"],
  "startTime": "1970-01-01T14:00:00Z",
  "endTime": "1970-01-01T17:00:00Z",
  "excludeForAllSlots": false,
  "includeForAllUnitSchedules": false,
  "assignedSchedules": ["sch_123","sch_456"],
  "healthUnitTaxId": "5002159961",
  "definedBy": "ops@baika"
}
```
- **Bloqueio pontual (janela de manutenção):**
```json
{
  "title": "Janela de Manutenção",
  "startDate": "2025-10-21T08:00:00Z",
  "endDate": "2025-10-21T10:00:00Z",
  "excludeForAllSlots": true,
  "includeForAllUnitSchedules": true,
  "healthUnitTaxId": "5002159961",
  "definedBy": "ops@baika"
}
```

---

## Modelo `ExcludeDay` (bloqueio do dia inteiro)

Usado para **remover um dia inteiro** da disponibilidade (feriados, recessos, inventários). Tem **prioridade** sobre `ExcludeRange`.

### Estrutura (conforme schema)

| Campo | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| `id` | `String @id @default(uuid())` | Sim | Identificador único. |
| `specificDate` | `DateTime? @db.Timestamptz` | Não | Data exacta (bloqueio pontual do **dia inteiro**). |
| `weekDays` | `WeekDays[]` | Não | Dias recorrentes (ex.: `[SUNDAY]` para encerrar aos domingos). |
| `typeOfRecurrence` | `TypeOfRecurrence?` | Não | Frequência para recorrência semanal/mensal/etc. |
| `rrule` | `String?` | Não | Regra iCalendar para padrões anuais (ex.: feriados). |
| `isActive` | `Boolean @default(true)` | Sim | Liga/desliga. |
| `healthUnitTaxId` | `String` | Sim | Escopo por unidade. |
| `schedules` | `Schedule_ExcludeDay_[] @relation("Day")` | Não | Relação M:N com **agendas** (quando a regra for por conjunto específico). |
| `title` | `String` | Sim | Título (ex.: “Natal”). |
| `reason` | `String?` | Não | Motivo/descrição. |
| `deletedAt` | `DateTime?` | Não | Soft-delete. |

> **Observação**: o schema não define os campos `includeForAllUnitSchedules/definedBy/updatedBy` para `ExcludeDay`. O escopo por agenda é definido via relação M:N (`schedules`). Caso seja necessário “todas as agendas da unidade”, convencionar **não associar** nenhuma agenda e tratar **via política** (ou criar um `ExcludeDay` por agenda com um procedimento auxiliar).

### Validações Essenciais
- Se usar `specificDate`, **não** usar simultaneamente uma `rrule` que resulte no mesmo dia (evitar duplicidade).  
- `isActive = false` ⇒ ignorar.  
- Quando usar `weekDays` com `typeOfRecurrence`, validar que o motor de geração entende a combinação (ex.: `WEEKLY` + `[SUNDAY]`).

### Exemplos
- **Feriado anual (todas as agendas da unidade, via política):**
```json
{
  "title": "Natal",
  "rrule": "FREQ=YEARLY;BYMONTH=12;BYMONTHDAY=25",
  "healthUnitTaxId": "5002159961"
}
```
- **Recesso pontual em dia específico para agendas selecionadas:**
```json
{
  "title": "Recesso Pós-Feriado",
  "specificDate": "2025-12-26T00:00:00Z",
  "healthUnitTaxId": "5002159961",
  "schedules": ["sch_123","sch_456"]
}
```

---

## Regras de Prioridade e Interação

1. **`ExcludeDay`** anula o **dia inteiro** (os *slots* do dia **não são gerados** ou são removidos).  
2. `ExcludeRange` aplica-se apenas **se o dia não estiver bloqueado** por `ExcludeDay`.  
3. Em colisão com horas **BOOKED/HELD**, **não remova**; apenas **marque como `BLOCKED`** e direcione o fluxo de **reagendamento/cancelamento**.  
4. `isActive = false` ⇒ nenhuma ação.  

---

## Recomendações de Implementação

- **UTC-only**: normalize todos os campos `DateTime`/`Time` para UTC.  
- **Engine de RRULE**: validar RRULE no *backend* (rejeitar strings inválidas).  
- **Escopo coerente**: aplicar as invariantes A/B do `ExcludeRange` no serviço antes de persistir.  
- **Idempotência**: **não** duplicar bloqueios equivalentes; prefira *upsert* por `(healthUnitTaxId, title, período)` se fizer sentido no teu domínio.
- **Auditoria**: utilizar `definedBy`, `updatedBy`, `updatedAt` (quando disponíveis) e **logs com `correlationId`** no processamento.

---

## Erros e Códigos sugeridos

| Código | Quando | Mensagem (exemplo) |
|------:|--------|---------------------|
| **400** | Payload malformado / RRULE inválida | `Invalid RRULE string` |
| **401** | Não autenticado | `Unauthorized` |
| **403** | Sem permissão para a unidade | `Forbidden for healthUnitTaxId=...` |
| **404** | Agenda(s) referida(s) inexistente(s) | `Schedule not found: sch_123` |
| **409** | Invariantes A/B violadas | `Ambiguous scope: set includeForAllUnitSchedules OR assignedSchedules` |
| **409** | Colisão com marcações existentes (política rígida) | `Cannot block occupied hour 2025-11-01T09:00Z` |
| **422** | Datas incoerentes | `startDate must be <= endDate` |
| **422** | Janelas inconsistentes | `startTime must be before endTime` |
| **422** | Combinação incoerente de recorrência | `typeOfRecurrence requires excludeFor or rrule` |
| **500** | Erro interno | `Failed to apply exclusions. correlationId=...` |

---

## Exemplo combinado (dia + janelas)

```json
{
  "excludeDays": [
    { "title": "Feriado Nacional", "rrule": "FREQ=YEARLY;BYMONTH=12;BYMONTHDAY=25", "healthUnitTaxId": "5002159961" },
    { "title": "Recesso", "specificDate": "2025-12-26T00:00:00Z", "healthUnitTaxId": "5002159961", "schedules": ["sch_123"] }
  ],
  "excludeRanges": [
    { "title": "Almoço", "typeOfRecurrence": "DAILY", "startTime": "1970-01-01T12:00:00Z", "endTime": "1970-01-01T13:00:00Z", "excludeForAllSlots": true, "includeForAllUnitSchedules": true, "healthUnitTaxId": "5002159961", "definedBy": "ops@baika" }
  ]
}
```

No exemplo acima:
- `ExcludeDay` remove **25/12** (via RRULE) e **26/12** (pontual) — **antes** de qualquer `ExcludeRange` ser aplicado.  
- `ExcludeRange` bloqueia **12:00–13:00** em **todos os schedules** da unidade, em **todos os dias** (enquanto ativo).
