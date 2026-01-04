# API Hospitalar — **Marcações (Appointments)**

> Todas as datas em **UTC** (ISO 8601). Autenticação via **Bearer Token**.  
> Paginação padrão: `page=1`, `limit=20`, `sortOrder=desc` (para campos de data).

---

## 4. MARCAÇÕES (APPOINTMENTS)

### 4.1. CRUD Básico
- **POST** `/appointments` — Criar nova marcação (modo **TIMED**, com `selectedHour`).
- **POST** `/appointments/capacity-mode` — Criar marcação em **modo capacidade** (sem horário; apenas `slotId`).
- **GET** `/appointments/{id}` — Obter marcação por ID.
- **PUT** `/appointments/{id}` — Atualizar marcação (substituição total).
- **DELETE** `/appointments/{id}` — Cancelar marcação (soft/cancel).

### 4.2. Consultas e Filtros por Escopo
- **GET** `/appointments` — Listar marcações (filtros abaixo).
- **GET** `/slots/{slotId}/appointments` — Marcações do slot.
- **GET** `/schedules/{scheduleId}/appointments` — Marcações da agenda.
- **GET** `/appointments/patient/{patientTaxId}` — Marcações do paciente.
- **GET** `/appointments/professional/{professionalTaxId}` — Marcações do profissional.
- **GET** `/appointments/status/{status}` — Marcações por status.

### 4.3. Gestão de Estados (transições)
- **PATCH** `/appointments/{id}/confirm` — Confirmar marcação.
- **PATCH** `/appointments/{id}/check-in` — Check-in do paciente.
- **PATCH** `/appointments/{id}/start-session` — Iniciar sessão.
- **PATCH** `/appointments/{id}/complete` — Completar consulta.
- **PATCH** `/appointments/{id}/no-show` — Registrar não comparecimento.
- **PATCH** `/appointments/{id}/expire` — Expirar marcação.

### 4.4. Cancelamentos Específicos
- **PATCH** `/appointments/{id}/cancel-by-patient`
- **PATCH** `/appointments/{id}/cancel-by-professional`
- **PATCH** `/appointments/{id}/cancel-by-system`

> **Nota de modo de operação**
>
> - **TIMED (por duração)**: requer `selectedHour` (deve existir no `hours[]` do `Slot` e estar `AVAILABLE`).  
> - **CAPACITY (dia como slot)**: **não** usa `selectedHour`; somente `slotId`. A capacidade é consumida via token interno.

---

## Filtros para **GET /appointments**

### 🎯 1) Básicos & Identificação
- **id** — ID específico da marcação  
- **slotId** — Slot específico  
- **scheduleId** — Agenda específica  
- **patientTaxId** — Paciente  
- **professionalTaxId** — Profissional  
- **paymentId** — ID de pagamento

### 🗓️ 2) Período & Data
- **selectedHourFrom**, **selectedHourTo** — Intervalo de data/hora da consulta  
- **selectedDate** — Data específica da consulta  
- **dateRange** — Faixa (ex.: `2024-12-01,2024-12-31`)  
- **createdAtFrom**, **createdAtTo** — Janela de criação

### ⚖️ 3) Status & Estado
- **status** — Um dos: `PENDING`, `CONFIRMED`, `CHECKED_IN`, `IN_SESSION`, `COMPLETED`, `CANCELLED_BY_PATIENT`, `CANCELLED_BY_PROFESSIONAL`, `CANCELLED_BY_SYSTEM`, `NO_SHOW`, `EXPIRED`, `FOLLOW_UP_REQUIRED`, `DOCUMENT_PENDING`
- **statusIn** — CSV de múltiplos status  
- **isActive** — Não canceladas/expiradas  
- **isCancelled** — Canceladas (qualquer subtipo)  
- **isCompleted** — Completas

### 👥 4) Participantes
- **patientTaxIds** — CSV de pacientes  
- **professionalTaxIds** — CSV de profissionais  
- **hasProfessional** — `true/false`  
- **cancelledByTaxId** — Quem cancelou  
- **hasCancellationInfo** — `true/false`

### 🏥 5) Serviço & Tipo
- **typeOfService** — `IN_PERSON`, `TELEMEDICINE`, `HOME`  
- **typeOfServiceIn** — CSV  
- **hasRoomLink** — `true/false`  
- **hasLocationRoom** — `true/false`  
- **hasNotes** — `true/false`  
- **hasPayment** — `true/false`  
- **priceCentsMin**, **priceCentsMax**

### ⏱️ 6) Temporais Avançados
- **today**, **tomorrow**, **thisWeek**, **nextWeek**, **past**, **future**, **upcoming**  
- **timeOfDay** — `morning (06:00–12:00)`, `afternoon (12:00–18:00)`, `evening (18:00–23:00)`  
- **hourRange** — ex.: `08:00-12:00`  
- **updatedAtFrom**, **updatedAtTo**  
- **statusChangedAtFrom**, **statusChangedAtTo**  
- **cancelledAtFrom**, **cancelledAtTo**

### 🔗 7) Relacionados a Agenda / Slot
- **healthUnitTaxId**, **specialityId**, **typeOfSchedule** (`CONSULTATION`, `VACCINE`, `EXAM`, `RETURN`)  
- **slotSpecificDate**, **slotWeekDay**, **slotIsClosed**, **slotInheritAllProfessionals**

### 🔁 8) Encaminhamentos & Remarcações
- **hasForwarding** — `true/false`  
- **forwardingStatus** — `PENDING`, `SENT`, `RECEIVED`, `ACCEPTED`, `REJECTED`, `COMPLETED`, `CANCELLED`  
- **forwardingDestination** — `INTERNAL`/`EXTERNAL`  
- **forwardingPriority** — `NORMAL`/`URGENT`/`SERIOUS`  
- **hasReschedule** — `true/false`  
- **rescheduleStatus** — (se aplicável)  
- **wasRescheduled** — `true/false`  
- **rescheduleRequestedBy** — TaxId

### 📑 9) Paginação, Ordenação, Seleção
- **page**, **limit**, **offset**  
- **sortBy** — `selectedHour`, `createdAt`, `updatedAt`, `statusChangedAt`, `patientTaxId`, `professionalTaxId`, `priceCents`  
- **sortOrder** — `asc` | `desc` (padrão: `desc` para datas)  
- **fields** — seleção de campos  
- **include** — relações: `slot`, `schedule`, `forwarding`, `reschedules`, `slot.schedule`

---

## 🧪 Exemplos

### 1) Marcações de um paciente
```http
GET /appointments?patientTaxId=BI12345&statusIn=PENDING,CONFIRMED&selectedHourFrom=2024-12-01&sortBy=selectedHour&sortOrder=asc
```

### 2) Agenda de um profissional
```http
GET /appointments?professionalTaxId=BI67890&selectedDate=2024-12-15&statusIn=CONFIRMED,CHECKED_IN&include=slot
```

### 3) Para confirmação
```http
GET /appointments?status=PENDING&selectedHourFrom=2024-12-01&selectedHourTo=2024-12-07&needsAttention=true
```

### 4) Estatísticas de cancelamento
```http
GET /appointments?isCancelled=true&cancelledAtFrom=2024-12-01&cancelledAtTo=2024-12-31&fields=id,cancelledAt,cancelledByTaxId
```

### 5) Teleconsultas agendadas
```http
GET /appointments?typeOfService=TELEMEDICINE&hasRoomLink=true&status=CONFIRMED&selectedHourFrom=2024-12-01
```

### 6) Pagamento pendente
```http
GET /appointments?hasPayment=false&status=PENDING&selectedHourFrom=2024-12-01&priceCentsMin=1
```

### 7) Não comparecimentos (por profissional)
```http
GET /appointments?status=NO_SHOW&selectedHourFrom=2024-12-01&selectedHourTo=2024-12-31&professionalTaxId=BI67890
```

### 8) Para hoje (manhã)
```http
GET /appointments?today=true&statusIn=CONFIRMED,CHECKED_IN&timeOfDay=morning&include=slot.schedule
```

### 9) Histórico completo do paciente
```http
GET /appointments?patientTaxId=BI12345&sortBy=selectedHour&sortOrder=desc&limit=50&include=slot.schedule,forwarding,reschedules
```

### 10) Encaminhamento pendente pós-consulta
```http
GET /appointments?hasForwarding=true&forwardingStatus=PENDING&status=COMPLETED&selectedHourFrom=2024-11-01
```

### 11) Remarcadas
```http
GET /appointments?wasRescheduled=true&selectedHourFrom=2024-12-01&include=reschedules
```

### 12) Dashboard administrativo (misto)
```http
GET /appointments?healthUnitTaxId=540102938&selectedHourFrom=2024-12-01&selectedHourTo=2024-12-31&include=slot.schedule&fields=id,selectedHour,status,patientTaxId,professionalTaxId,typeOfService
```

### 13) Expiradas
```http
GET /appointments?status=EXPIRED&selectedHourFrom=2024-12-01&updatedAtFrom=2024-12-01
```

### 14) Por faixa etária
```http
GET /appointments?patientAgeMin=18&patientAgeMax=65&status=COMPLETED&selectedHourFrom=2024-11-01
```

### 15) Busca complexa
```http
GET /appointments?healthUnitTaxId=540102938&specialityId=CARDIOLOGY&typeOfService=IN_PERSON&statusIn=CONFIRMED,CHECKED_IN&selectedDate=2024-12-15&timeOfDay=afternoon&hasProfessional=true&include=slot&sortBy=selectedHour&sortOrder=asc
```

---

## ⚙️ Estrutura de Resposta (Paginada + Stats)

```json
{
  "data": [
    {
      "id": "appt-123",
      "selectedHour": "2024-12-15T10:00:00Z",
      "patientTaxId": "BI12345",
      "patientAge": 35,
      "professionalTaxId": "BI67890",
      "typeOfService": "TELEMEDICINE",
      "status": "CONFIRMED",
      "statusChangedAt": "2024-12-14T15:30:00Z",
      "priceCents": 5000,
      "currency": "USD",
      "roomLink": "https://meet.example.com/room-123",
      "notes": "Consulta de rotina",
      "slot": {
        "id": "slot-456",
        "specificDate": "2024-12-15T00:00:00Z",
        "weekDay": "MONDAY",
        "isClosed": false
      },
      "schedule": {
        "healthUnitTaxId": "540102938",
        "specialityId": "CARDIOLOGY",
        "typeOfSchedule": "CONSULTATION"
      },
      "forwarding": [],
      "reschedules": []
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
  "stats": {
    "totalAppointments": 150,
    "byStatus": {
      "PENDING": 15,
      "CONFIRMED": 45,
      "COMPLETED": 60,
      "CANCELLED": 30
    },
    "byTypeOfService": {
      "IN_PERSON": 100,
      "TELEMEDICINE": 50
    }
  }
}
```

---

## ✅ Validações Importantes
- **Enums**: devem corresponder ao schema (`AppointmentStatus`, `TypeOfService`, etc.).  
- **TIMED**: `selectedHour` obrigatório e pertencente ao `Slot.hours` **livre** no momento da reserva; `slot.isClosed=false`.  
- **CAPACITY**: não enviar `selectedHour`; apenas `slotId` (a capacidade é consumida como token).  
- **Autorização**: filtrar resultados por escopo do usuário (LGPD/PHI).  
- **Limites**: `limit` máximo 100.  
- **Performance**: usar índices por `status/selectedHour/professionalTaxId/patientTaxId`.  

## 🧱 Códigos de Erro Comuns
| Código | Quando | Mensagem |
|---|---|---|
| 400 | Parâmetros inválidos / datas mal formatadas | `Invalid parameters` |
| 401 | Sem credenciais válidas | `Unauthorized` |
| 403 | Sem permissão de leitura/ação | `Forbidden` |
| 404 | Recurso inexistente (`appointment/slot/schedule`) | `Not found` |
| 409 | Conflito de reserva (hora já ocupada / slot fechado) | `Hour already booked / Slot is closed` |
| 409 | Modo incorreto (enviar `selectedHour` no modo capacidade) | `Capacity mode does not accept selectedHour` |
| 422 | Regra de negócio violada (status flow inválido, enum inválido) | `Unprocessable entity` |
| 500 | Erro interno | `Internal server error` |

---

## 🧭 Notas de Implementação
- **Idempotência** em `POST` via `Idempotency-Key`.  
- **Concorrência otimista** com `ETag`/`If-Match` em `PATCH` críticos.  
- **Auditoria**: registrar `statusChangedAt`, `cancelledAt`, `cancelledByTaxId`.  
- **Telemedicina**: validar `roomLink`/slot `typeOfService` compatível com a marcação.  
- **Observabilidade**: `correlationId` em logs e respostas em operações longas.