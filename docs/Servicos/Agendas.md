# 1) Gestão de Agendas (Schedules) — Endpoints e Filtros (Compatível com Schemas Atuais)

> Todas as datas em **UTC** (ISO 8601). Paginação padrão: `page=1`, `limit=20` (máx. 100).  
> **Enums (do schema):**  
> - **TypeOfService:** `IN_PERSON` \| `TELEMEDICINE` \| `HOME`  
> - **TypeOfRecurrence:** `NONE` \| `DAILY` \| `WEEKLY` \| `MONTHLY` \| `QUARTERLY` \| `SEMIANNUALLY` \| `ANNUALLY` \| `CUSTOM`  
> - **TypeOfSchedule:** `CONSULTATION` \| `VACCINE` \| `EXAM` \| `RETURN`

---

## 1. GESTÃO DE AGENDAS (SCHEDULES)

### 1.1. CRUD Básico
- **POST** `/schedules` — Criar nova agenda  
- **GET** `/schedules/{id}` — Obter agenda por ID  
- **PUT** `/schedules/{id}` — Atualizar agenda completa  
- **PATCH** `/schedules/{id}` — Atualização parcial  
- **DELETE** `/schedules/{id}` — *Soft delete* (marca `deletedAt`)  
- **GET** `/schedules` — Listar agendas com filtros

### 1.2. Operações Específicas
- **GET** `/schedules/health-unit/{healthUnitTaxId}` — Agendas por unidade  
- **GET** `/schedules/speciality/{specialityId}` — Agendas por especialidade  
- **GET** `/schedules/professional/{professionalTaxId}` — Agendas que **incluem** o profissional em `availableProfessionalTaxIds`  
  > **Nota:** a disponibilidade real do profissional é aplicada na **geração de slots** cruzando `ProfessionalWorkTime`.
- **POST** `/schedules/{id}/clone` — Clonar agenda  
- **POST** `/schedules/{id}/restore` — Restaurar agenda *soft-deleted*

---

## Filtros para **GET /schedules**

### 1) Básicos

#### 1.1. Identificação
- `healthUnitTaxId` — Filtra por unidade de saúde  
- `specialityId` — Filtra por especialidade  
- `createdBy` — Usuário que criou a agenda  

#### 1.2. Período de Vigência
- `startDateFrom` — Data de início a partir de  
- `startDateTo` — Data de início até  
- `endDateFrom` — Data de término a partir de  
- `endDateTo` — Data de término até  
- `activeOn` — Agendas **ativas** numa data específica *(derivado)*  
- `activeBetween` — Agendas **ativas** num período *(derivado; overlap)*  

#### 1.3. Tipo de Serviço/Agenda
- `typeOfService` — `IN_PERSON` \| `TELEMEDICINE` \| `HOME`  
- `typeOfSchedule` — `CONSULTATION` \| `VACCINE` \| `EXAM` \| `RETURN`

---

### 2) Configuração

#### 2.1. Recorrência e Dias
- `typeOfRecurrence` — `NONE` \| `DAILY` \| `WEEKLY` \| `MONTHLY` \| `QUARTERLY` \| `SEMIANNUALLY` \| `ANNUALLY` \| `CUSTOM`  
- `weekDays` — Dias da semana (CSV: `MONDAY,FRIDAY,...`)  
- `hasRecurrence` — **derivado** (`typeOfRecurrence != NONE`)

#### 2.2. Modo de Operação (**derivado**)
- `mode` — `TIMED` \| `CAPACITY`  
  - `TIMED` ⇒ `appointmentDurationInMinutes > 0`  
  - `CAPACITY` ⇒ `appointmentDurationInMinutes IS NULL` **e** `maxAppointmentsPerSlot > 0`
- `hasAppointmentDuration` — agendas com duração definida *(derivado)*  
- `hasMaxAppointments` — agendas com capacidade máxima *(derivado)*  

#### 2.3. Herança e Geração
- `eachSlotInheritAllScheduleProfessionals` — `true/false`  
- `automaticallyGenerateSlots` — `true/false`

---

### 3) Profissionais
- `professionalTaxId` — Agenda **inclui** o profissional (em `availableProfessionalTaxIds`)  
- `availableProfessionalTaxIds` — CSV de profissionais atribuídos  
- `hasProfessionals` — **derivado** (tem profissionais atribuídos)

---

### 4) Estado

#### 4.1. Estado da Agenda *(derivado)*
- `isActive` — `deletedAt IS NULL` **e** `startDate ≤ now()` **e** (`endDate IS NULL` **ou** `now() ≤ endDate`)  
- `hasExclusions` — possui `ExcludeRange` **ou** `ExcludeDay` relacionados  
- `hasSlots` — possui `Slots` gerados

#### 4.2. Datas de Criação/Modificação
- `createdAtFrom`, `createdAtTo`  
- `updatedAtFrom`, `updatedAtTo`

---

### 5) Exclusões
- `hasExcludeRanges` — com exclusões por **intervalo**  
- `hasExcludeDays` — com exclusões por **dia**  
- `exclusionType` — livre/convencionado: `RANGE` \| `DAY` \| `BOTH`

> **Nota:** As relações são **M:N** (`Schedules_ExcludeRange_`, `Schedule_ExcludeDay_`). Uma exclusão pode aplicar-se a várias agendas e vice-versa.

---

### 6) Paginação e Ordenação
- `page` — Página (padrão: 1)  
- `limit` — Itens/página (padrão: 20, **máx.: 100**)  
- `offset` — Alternativa à paginação por página

- `sortBy` — `startDate` \| `endDate` \| `createdAt` \| `updatedAt` \| `healthUnitTaxId` \| `specialityId`  
- `sortOrder` — `asc` \| `desc` (padrão: `desc`)

---

### 7) Busca
- `search` — Busca textual (definir campos suportados; ex.: `specialityId`, `healthUnitTaxId`)  
- `q` — Query geral (alias de busca textual)

---

### 8) Deletados
- `includeDeleted` — Incluir *soft-deleted* (padrão: `false`)  
- `onlyDeleted` — Retornar **apenas** *soft-deleted*

---

## 📋 Exemplos de Uso

**Ex. 1 — Agendas presenciais ativas de uma unidade**
```http
GET /schedules?healthUnitTaxId=540102938&typeOfService=IN_PERSON&isActive=true
```

**Ex. 2 — Agendas ativas por especialidade, com geração automática**
```http
GET /schedules?specialityId=CARDIOLOGY&activeOn=2025-01-15&automaticallyGenerateSlots=true
```

**Ex. 3 — Agendas que incluem um profissional (recorrência semanal)**
```http
GET /schedules?professionalTaxId=BI12345&typeOfRecurrence=WEEKLY&weekDays=MONDAY,WEDNESDAY,FRIDAY
```

**Ex. 4 — Paginação e ordenação**
```http
GET /schedules?healthUnitTaxId=540102938&page=1&limit=10&sortBy=startDate&sortOrder=asc
```

**Ex. 5 — Modo TIMED (derivado)**
```http
GET /schedules?healthUnitTaxId=540102938&specialityId=DERMATOLOGY&typeOfService=TELEMEDICINE&startDateFrom=2025-01-01&activeOn=2025-01-20&mode=TIMED
```

**Ex. 6 — Com exclusões**
```http
GET /schedules?hasExcludeRanges=true&hasExcludeDays=true&isActive=true
```

---

## ⚙️ Resposta Paginada
```json
{
  "data": [
    {
      "id": "c1f7a9a0-4c3f-4e2a-8b1d-9e0d7f6a3b5c",
      "healthUnitTaxId": "540102938",
      "specialityId": "DERMATOLOGY",
      "weekDays": ["MONDAY", "WEDNESDAY", "FRIDAY"],
      "startTime": "2025-01-01T08:00:00.000Z",
      "endTime": "2025-01-01T17:00:00.000Z",
      "appointmentDurationInMinutes": 30,
      "maxAppointmentsPerSlot": null,
      "typeOfRecurrence": "WEEKLY",
      "startDate": "2025-01-01T00:00:00.000Z",
      "endDate": "2025-12-31T23:59:59.000Z",
      "typeOfSchedule": "CONSULTA",
      "typeOfService": "TELECONSULTA",
      "advanceCancellationInHours": 24,
      "deadlineForSlotBookingInHours": 2,
      "availableProfessionalTaxIds": ["PROF001", "PROF002"],
      "automaticallyGenerateSlots": true,
      "eachSlotInheritAllScheduleProfessionals": true,
      "applyGenderRestrictions": "NONE",
      "autoPublishSlots": false,
      "requireSupervisorApproval": true,
      "defaultPublishBufferHours": 4,
      "defaultReviewTTLHours": 48,
      "createdBy": "user_admin_01",
      "createdAt": "2024-10-28T10:00:00.000Z",
      "updatedAt": "2024-10-28T11:30:00.000Z",
      "deletedAt": null
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": false
  },
  "filters": {
    "applied": { },
    "available": { }
  }
}
```

---

## 🔍 Validações Importantes
- **Datas:** ISO 8601 (sempre UTC).  
- **Enums:** usar **exatamente** os valores do schema (ver cabeçalho).  
- **Limites:** `limit ≤ 100`.  
- **Permissões:** filtros por unidade/especialidade podem ser restringidos por RBAC/ABAC.  
- **Performance:** `hasSlots/hasExclusions` → usar `EXISTS`/índices; `mode` é **derivado** (condições internas).

---

### Observações de Implementação
- `isActive`/`activeOn`/`activeBetween` são **derivados** de `startDate/endDate/deletedAt` (não existem como colunas).  
- `mode TIMED/CAPACITY` é **derivado** de `appointmentDurationInMinutes` e `maxAppointmentsPerSlot`.  
- Disponibilidade real do profissional depende de **ProfessionalWorkTime** e é aplicada na **geração/validação de slots** (não neste endpoint).
