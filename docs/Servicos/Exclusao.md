# API Hospitalar — **Exclusões** (Exclude Ranges/Days)

> Todas as datas/horas em **UTC** (ISO 8601).  
> Autenticação: **Bearer Token**. Autorização: **RBAC/ABAC** conforme perfis.  
> **Nota de modelo**: `ExcludeRange.startTime`/`endTime` são `DateTime?` (UTC) usados sobretudo para **padrões recorrentes** (hora do dia). Para exclusões **pontuais**, use `startDate`/`endDate` (UTC).  
> **Escopo**: `ExcludeRange` pode atuar ao nível da **unidade** (`healthUnitTaxId`) com `includeForAllUnitSchedules=true`, e/ou ser associado a agendas específicas.

---

## 1. Endpoints de Exclusões

### 1.1. ExcludeRange
- **POST** `/schedules/{scheduleId}/exclude-ranges` — Criar exclusão por **intervalo**
- **GET** `/schedules/{scheduleId}/exclude-ranges` — Listar exclusões por intervalo de uma agenda
- **GET** `/exclude-ranges/{id}` — Obter exclusão específica
- **PUT** `/exclude-ranges/{id}` — Atualizar exclusão
- **DELETE** `/exclude-ranges/{id}` — Remover exclusão (soft delete recomendado)
- **PATCH** `/exclude-ranges/{id}/toggle` — Ativar/Desativar exclusão
- **GET** `/exclude-ranges` — Listagem **global** (por unidade, escopo e filtros)

**Campos relevantes no modelo (resumo):**
- `healthUnitTaxId: string` — escopo na unidade
- `includeForAllUnitSchedules: boolean` — aplica-se a **todas** as agendas da unidade
- `assignedSchedules: Schedules[]` — associação explícita a agendas (quando **não** global)
- `startTime/endTime: DateTime?` — janela horária (recorrente; usar componente de hora)
- `startDate/endDate: DateTime?` — janela de datas (pontual/intervalar)
- `typeOfRecurrence: TypeOfRecurrence` — `NONE/DAILY/WEEKLY/MONTHLY/...`
- `excludeFor: WeekDays[]` — dias alvo na recorrência (ignorado se `includeForAllUnitSchedules=true`)
- `excludeForSpecificDates: DateTime[]` — datas específicas (usa quando `typeOfRecurrence=NONE`)
- `rrule: string?` — regra iCalendar opcional
- `isActive: boolean` — status da exclusão
- `title/reason` — metadados descritivos

---

### 1.2. ExcludeDay
- **POST** `/schedules/{scheduleId}/exclude-days` — Criar exclusão por **dia**
- **GET** `/schedules/{scheduleId}/exclude-days` — Listar exclusões por dia de uma agenda
- **GET** `/exclude-days/{id}` — Obter exclusão específica
- **PUT** `/exclude-days/{id}` — Atualizar exclusão
- **DELETE** `/exclude-days/{id}` — Remover exclusão (soft delete recomendado)
- **PATCH** `/exclude-days/{id}/toggle` — Ativar/Desativar exclusão
- **GET** `/exclude-days` — Listagem **global** (por unidade, escopo e filtros)

**Campos relevantes no modelo (resumo):**
- `healthUnitTaxId: string` — escopo na unidade
- `schedules: Schedules[]` — associação explícita a agendas
- `specificDate: DateTime?` — dia exato (pontual)
- `weekDays: WeekDays[]` + `typeOfRecurrence?: TypeOfRecurrence` — bloqueios **recorrentes**
- `rrule: string?` — regra iCalendar opcional
- `isActive: boolean` — status da exclusão
- `title/reason` — metadados descritivos

---

## 2. Filtros — **Exclude Range**

> Endpoints: **GET** `/exclude-ranges` e **GET** `/schedules/{scheduleId}/exclude-ranges`

### 2.1. Básicos & Escopo
- `scheduleId` — filtra por agenda específica (ignorado se `includeForAllUnitSchedules=true` e busca global)
- `healthUnitTaxId` — filtra por unidade
- `includeForAllUnitSchedules` — `true/false`
- `isActive` — `true/false`
- `title`, `reason` — busca textual (LIKE/ILIKE)

### 2.2. Tipo & Recorrência
- `typeOfRecurrence` — `NONE/DAILY/WEEKLY/MONTHLY/...`
- `excludeFor` — CSV de `WeekDays` (ex.: `MONDAY,FRIDAY`)
- `hasSpecificDates` — `true/false` (se `excludeForSpecificDates` tem valores)
- `hasRrule` — `true/false`
- `rrule` — busca exata/substring da RRULE

### 2.3. Janelas Temporais
- **Pontuais (datas):**
  - `startDateFrom`, `startDateTo`, `endDateFrom`, `endDateTo`
  - `activeOnDate` — ativa numa data (interseção com o período)
  - `activeBetween=from,to` — ativa dentro de um intervalo
- **Recorrentes (hora do dia):**
  - `startTimeFrom`, `startTimeTo`, `endTimeFrom`, `endTimeTo`  
    > Como `startTime/endTime` são `DateTime`, considerar **apenas** o **componente de hora** (00:00 do dia base).

### 2.4. Auditoria/Estado
- `createdAtFrom`, `createdAtTo`, `updatedAtFrom`, `updatedAtTo`
- `includeDeleted` (`false` por padrão), `onlyDeleted`

### 2.5. Paginação/Ordenação/Busca
- `page` (padrão: 1), `limit` (padrão: 20, máx.: 100), `offset`
- `sortBy`: `title`, `createdAt`, `updatedAt`, `startDate`, `endDate`, `startTime`, `endTime`
- `sortOrder`: `asc|desc` (padrão: `desc` para datas)
- `search`, `q`: busca livre em `title/reason`

**Exemplos**
```http
GET /exclude-ranges?healthUnitTaxId=540102938&includeForAllUnitSchedules=true&isActive=true
GET /schedules/sch-123/exclude-ranges?typeOfRecurrence=DAILY&excludeFor=MONDAY,FRIDAY&isActive=true
GET /exclude-ranges?title=Almoço&startTimeFrom=12:00&endTimeTo=13:00&isActive=true
GET /exclude-ranges?hasSpecificDates=true&activeBetween=2025-01-01,2025-01-31
```

---

## 3. Filtros — **Exclude Day**

> Endpoints: **GET** `/exclude-days` e **GET** `/schedules/{scheduleId}/exclude-days`

### 3.1. Básicos & Escopo
- `scheduleId`, `healthUnitTaxId`
- `isActive`
- `title`, `reason`

### 3.2. Tipo & Recorrência
- `typeOfRecurrence`
- `hasRrule`, `rrule`
- `hasSpecificDate` — `true/false`
- `hasWeekDays` — `true/false`

### 3.3. Datas específicas & Padrões
- **Pontual**: `specificDateFrom`, `specificDateTo`, `specificDateOn`, `specificDateBetween`
- **Recorrente**: `weekDays` (CSV), `month`, `year`, `dayOfMonth` (útil para relatórios)

### 3.4. Auditoria/Estado
- `createdAtFrom`, `createdAtTo`, `updatedAtFrom`, `updatedAtTo`
- `includeDeleted`, `onlyDeleted`

### 3.5. Paginação/Ordenação/Busca
- `page`, `limit`, `offset`
- `sortBy`: `title`, `createdAt`, `updatedAt`, `specificDate`, `weekDays`
- `sortOrder`: `asc|desc`
- `search`, `q`

**Exemplos**
```http
GET /exclude-days?healthUnitTaxId=540102938&isActive=true&hasWeekDays=true&typeOfRecurrence=WEEKLY
GET /schedules/sch-999/exclude-days?title=Feriado&specificDateFrom=2025-12-20&specificDateTo=2026-01-10
GET /exclude-days?hasRrule=true&rrule=FREQ=YEARLY;BYMONTH=12;BYMONTHDAY=25
```

---

## 4. Estruturas de Resposta (200)

### 4.1. ExcludeRange
```json
{
  "data": [
    {
      "id": "exrng-123",
      "healthUnitTaxId": "540102938",
      "includeForAllUnitSchedules": true,
      "assignedSchedules": ["sch-456", "sch-789"],
      "title": "Intervalo de Almoço",
      "reason": "Pausa operacional",
      "typeOfRecurrence": "DAILY",
      "excludeFor": ["MONDAY","FRIDAY"],
      "excludeForSpecificDates": [],
      "startTime": "1970-01-01T12:00:00Z",
      "endTime": "1970-01-01T13:00:00Z",
      "startDate": null,
      "endDate": null,
      "rrule": null,
      "isActive": true,
      "createdAt": "2025-01-15T10:30:00Z",
      "updatedAt": "2025-01-15T10:30:00Z",
      "deletedAt": null
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 12, "totalPages": 1, "hasNext": false, "hasPrev": false }
}
```

### 4.2. ExcludeDay
```json
{
  "data": [
    {
      "id": "exday-001",
      "healthUnitTaxId": "540102938",
      "scheduleIds": ["sch-123"],
      "title": "Feriado Nacional",
      "reason": "Natal",
      "typeOfRecurrence": "YEARLY",
      "weekDays": [],
      "specificDate": null,
      "rrule": "FREQ=YEARLY;BYMONTH=12;BYMONTHDAY=25",
      "isActive": true,
      "createdAt": "2025-01-10T12:00:00Z",
      "updatedAt": "2025-01-10T12:00:00Z",
      "deletedAt": null
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 8, "totalPages": 1, "hasNext": false, "hasPrev": false }
}
```

---

## 5. Validações & Regras (resumo)

### 5.1. ExcludeRange
- **Coerência**: `startDate <= endDate` (quando ambos), `startTime < endTime` (recorrente)
- **Escopo**: exigir **um** dos cenários:
  - `includeForAllUnitSchedules=true` **OU**
  - `assignedSchedules` **não vazia**
- **Recorrência**:
  - `typeOfRecurrence !== NONE` ⇒ `excludeFor` **não vazio** (salvo quando `includeForAllUnitSchedules=true`)
  - `typeOfRecurrence === NONE` ⇒ usar `excludeForSpecificDates` **OU** `startDate/endDate`
- **RRULE**: se informado, deve ser válida (RFC 5545) e **coerente** com os demais campos

### 5.2. ExcludeDay
- **Pontual**: `specificDate` definido **OR**
- **Recorrente**: `weekDays` **+** `typeOfRecurrence`
- **RRULE**: se informado, válido (RFC 5545), e coerente com o modo (pontual/recorrente)

---

## 6. Erros Padrão

| Código | Onde | Quando | Mensagem (exemplo) |
|---|---|---|---|
| 400 | Ranges/Days | Datas inválidas (`from>to`, `startTime>=endTime`) | `Invalid date/time range` |
| 401 | Ambos | Falta de credenciais | `Unauthorized` |
| 403 | Ambos | Sem permissão para a agenda/unidade | `Forbidden` |
| 404 | Ambos | Recurso inexistente (id, scheduleId) | `Not Found` |
| 409 | Ranges | Conflito de escopo (nem global nem associado), sobreposição impeditiva | `Conflict` |
| 412 | Ambos | Falha de ETag/If-Match (concorrência otimista) | `Precondition Failed` |
| 422 | Ambos | RRULE inválida, enum inválido, recorrência sem dias | `Unprocessable Entity` |
| 500 | Ambos | Erro interno não tratado | `Internal Server Error` |

---

## 7. Observações de Implementação

- **Hora recorrente** (`startTime`/`endTime`) deve ser interpretada considerando **apenas o relógio** (HH:mm:ss) no cálculo diário; grave como `DateTime` em UTC (por convenção usar `1970-01-01THH:mm:ssZ`).  
- **Integração com geração de slots**: prioridade **ExcludeDay > ExcludeRange**. Nunca remova timeboxes **BOOKED/HELD**; marque como `BLOCKED` quando aplicável.  
- **Admin**: permitir exclusões **por unidade** com `includeForAllUnitSchedules=true` e auditoria (`updatedBy`, `deletedBy`).  
- **Performance**: indexar `healthUnitTaxId`, `updatedAt`, `deletedAt`, `typeOfRecurrence`. Paginar consultas globais.

---

## 8. Exemplos adicionais

# ExcludeRange global por unidade (almoço diário 12–13)
```http
POST /schedules/any/exclude-ranges
```
```json
{
  "healthUnitTaxId": "540102938",
  "includeForAllUnitSchedules": true,
  "typeOfRecurrence": "DAILY",
  "excludeFor": ["MONDAY","TUESDAY","WEDNESDAY","THURSDAY","FRIDAY"],
  "startTime": "1970-01-01T12:00:00Z",
  "endTime": "1970-01-01T13:00:00Z",
  "title": "Intervalo de almoço",
  "isActive": true
}
```

# ExcludeDay recorrente (todas as segundas) numa agenda específica
```http
POST /schedules/sch-123/exclude-days
```
```json
{
  "healthUnitTaxId": "540102938",
  "weekDays": ["MONDAY"],
  "typeOfRecurrence": "WEEKLY",
  "title": "Fecho de segunda",
  "isActive": true
}
```

---

> Este documento reflete os **novos campos e comportamentos** dos modelos `ExcludeRange` e `ExcludeDay`, alinhado aos filtros e à integração com a lógica de geração de slots.
