# API Hospitalar — Endpoints (Relatórios, Lote e Administração)

> Todas as datas em **UTC** (ISO 8601). Padrões: `page=1`, `limit=20`, `sortOrder=desc`.  
> Autenticação: Bearer Token. Autorização: RBAC/ABAC conforme perfis.

---

## 7. RELATÓRIOS E ANALÍTICAS

### 7.1. Relatórios de Agendamento

#### GET `/reports/appointments/daily` — Relatório diário de marcações
**Objetivo:** Volume diário de marcações + breakdown por status/serviço/profissional.

**Query params**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `from` | date | sim | Início do período (incl.) |
| `to` | date | sim | Fim do período (incl.) |
| `healthUnitTaxId` | string | não | Filtra por unidade |
| `specialityId` | string | não | Filtra por especialidade |
| `professionalTaxId` | string | não | Filtra por profissional |
| `typeOfService` | enum | não | IN_PERSON/TELEMEDICINE/HOME |
| `statusIn` | csv | não | Lista de `AppointmentStatus` |

**Resposta (200)**
```json
{
  "granularity": "DAILY",
  "from": "2025-01-01",
  "to": "2025-01-31",
  "series": [
    {"day":"2025-01-01","total":85,"byStatus":{"CONFIRMED":58,"COMPLETED":17,"CANCELLED_BY_PATIENT":10}},
    {"day":"2025-01-02","total":92,"byStatus":{"CONFIRMED":60,"COMPLETED":20,"NO_SHOW":12}}
  ],
  "totals": {"total": 2450, "byService":{"IN_PERSON":1600,"TELEMEDICINE":700,"HOME":150}}
}
```

**Erros**

| Código | Quando | Mensagem |
|---|---|---|
| 400 | from/to inválidos ou from>to | Invalid date range |
| 403 | Sem permissão de leitura | Forbidden |
| 422 | Enum/filters inválidos | Invalid filter value |
| 500 | Falha inesperada | Internal server error |

---

#### GET `/reports/appointments/monthly` — Relatório mensal
**Objetivo:** Agregação mensal (útil para tendências).

**Query params:** iguais ao diário, mas com janela longa (até 24 meses).

**Resposta (200)**
```json
{
  "granularity":"MONTHLY",
  "series":[
    {"month":"2024-12","total":1930,"byStatus":{"COMPLETED":1100,"NO_SHOW":120}},
    {"month":"2025-01","total":2050,"byStatus":{"COMPLETED":1180,"NO_SHOW":140}}
  ]
}
```

**Erros:** mesmos do diário.

---

#### GET `/reports/occupancy` — Taxa de ocupação
**Objetivo:** % de ocupação de slots/horas no período.

**Query params**

| Nome | Tipo | Descrição |
|---|---|---|
| `from`,`to` | date | Janela |
| `groupBy` | enum | DAY/WEEK/MONTH/PROFESSIONAL/SCHEDULE |
| `healthUnitTaxId`,`specialityId`,`professionalTaxId` | string | Filtros |
| `mode` | enum | TIMED (por hour-box) / CAPACITY (por tokens) |

**Resposta (200)**
```json
{
  "groupBy":"PROFESSIONAL",
  "mode":"TIMED",
  "items":[
    {"key":"BI12345","name":"Dra. Costa","capacity":520,"consumed":442,"occupancy":0.85},
    {"key":"BI67890","name":"Dr. Silva","capacity":480,"consumed":360,"occupancy":0.75}
  ]
}
```

**Erros**

| Código | Quando |
|---|---|
| 400 | groupBy/mode inválidos |
| 403 | Sem permissão |
| 500 | Falha inesperada |

---

#### GET `/reports/cancellation-rates` — Taxas de cancelamento
**Objetivo:** % canceladas vs confirmadas, por causa/origem (paciente, profissional, sistema).

**Query params:** from, to, filtros de unidade/especialidade/profissional/serviço.

**Resposta (200)**
```json
{
  "from":"2025-01-01","to":"2025-01-31",
  "rate":0.12,
  "bySource":{"PATIENT":0.07,"PROFESSIONAL":0.03,"SYSTEM":0.02},
  "byDay":[{"day":"2025-01-10","rate":0.15},{"day":"2025-01-11","rate":0.08}]
}
```

**Erros:** 400/403/500.

---

#### GET `/reports/no-show-rates` — Taxa de não comparecimento
**Objetivo:** % NO_SHOW por período e cortes (profissional, serviço, hora do dia).

**Query params:** from,to, groupBy (DAY|HOUR|PROFESSIONAL|SERVICE).

**Resposta (200)**
```json
{
  "groupBy":"HOUR",
  "items":[
    {"hour":"08","total":200,"noShow":22,"rate":0.11},
    {"hour":"14","total":180,"noShow":30,"rate":0.1667}
  ]
}
```

**Erros:** 400/403/500.

---

### 7.2. Métricas de Performance

#### GET `/reports/professional-performance`
**Objetivo:** KPIs por profissional (tempo médio, conversão, satisfação — se disponível).

**Query params**

| Nome | Tipo | Descrição |
|---|---|---|
| `from`,`to` | date | Período |
| `professionalTaxId` | string/csv | Um ou mais |
| `metrics` | csv | avgTIMED,completedRate,noShowRate,cancelRate |

**Resposta (200)**
```json
{
  "items":[
    {"professionalTaxId":"BI12345","avgTIMEDMin":27,"completedRate":0.82,"noShowRate":0.08,"cancelRate":0.10}
  ]
}
```

**Erros:** 400/403/422/500.

---

#### GET `/reports/schedule-utilization`
**Objetivo:** Utilização de agendas (por Schedules.id).

**Query params:** from,to,scheduleId (csv).

**Resposta (200)**
```json
{
  "items":[
    {"scheduleId":"sch-1","capacity":960,"used":720,"utilization":0.75},
    {"scheduleId":"sch-2","capacity":640,"used":448,"utilization":0.70}
  ]
}
```

---

#### GET `/reports/slot-efficiency`
**Objetivo:** Eficiência dos slots (tempo ocioso, distribuição de picos, impacto de exclusões).

**Query params:** from,to,healthUnitTaxId,specialityId.

**Resposta (200)**
```json
{
  "efficiencyIndex":0.88,
  "avgIdleBoxesPerDay":3.2,
  "peakHours":["09:00","10:00","14:00"],
  "excludeImpact":{"boxesRemoved":180,"daysBlocked":12}
}
```

**Erros:** 400/403/500.

---

## 9. OPERAÇÕES EM LOTE

### 9.1. Operações Massivas — Conceitos Gerais
- **Idempotência:** header `Idempotency-Key` (UUID) obrigatório; repetições devolvem o mesmo resultado.
- **Dry-run:** query `dryRun=true` para validar sem gravar.
- **Política de falhas:** `allOrNothing=true` (transacção única) ou agregação de `successes[]`/`failures[]`.

---

#### POST `/schedules/bulk` — Criar múltiplas agendas
**Body (exemplo)**
```json
{
  "allOrNothing": false,
  "items": [
    {
      "healthUnitTaxId": "540102938",
      "specialityId": "CARDIOLOGY",
      "weekDays": ["MONDAY","WEDNESDAY"],
      "startTime": "2025-02-01T08:00:00Z",
      "endTime": "2025-02-01T16:00:00Z",
      "appointmentTIMEDInMinutes": 20,
      "typeOfRecurrence": "WEEKLY",
      "startDate": "2025-02-01T00:00:00Z",
      "typeOfSchedule": "CONSULTATION",
      "typeOfService": "IN_PERSON",
      "availableProfessionalTaxIds": ["BI123","BI456"],
      "automaticallyGenerateSlots": true,
      "createdBy": "ops"
    }
  ]
}
```

**Resposta (207 Multi-Status quando allOrNothing=false)**
```json
{
  "successes":[{"index":0,"scheduleId":"sch-abc"}],
  "failures":[]
}
```

**Erros**

| Código | Quando |
|---|---|
| 207 | Resultados mistos |
| 201 | Tudo criado (quando allOrNothing=true) |
| 409 | Duplicidade lógica |
| 422 | Payload inválido |
| 400 | Sem items |
| 401/403 | Sem credenciais/permissões |
| 500 | Falha global |

---

#### POST `/slots/bulk-generate` — Gerar slots em lote
**Objetivo:** gerar slots para muitos `scheduleId` e/ou para um período grande (assíncrono recomendado).

**Body**
```json
{
  "scheduleIds": ["sch-1","sch-2"],
  "from": "2025-02-01T00:00:00Z",
  "to": "2025-03-31T00:00:00Z",
  "mode": "ASYNC"  // ASYNC|SYNC (SYNC limitado por janela curta)
}
```

**Resposta**
- `202 Accepted` (ASYNC): `{ "jobId":"job-789" }`  
- `200 OK` (SYNC curto): relatório com contagem/erros por agenda.

**Erros**

| Código | Quando |
|---|---|
| 404 | scheduleId inexistente |
| 409 | Job já em curso para a mesma janela |
| 422 | from>to ou janela longa em SYNC |
| 503 | Worker indisponível |

---

#### POST `/appointments/bulk` — Criar marcações em lote
**Body**
```json
{
  "allOrNothing": false,
  "items": [
    {
      "slotId": "slot-1",
      "selectedHour": "2025-02-10T10:00:00Z",  // omitir no modo capacidade
      "patientTaxId": "BI999",
      "professionalTaxId": "BI123",
      "typeOfService": "IN_PERSON"
    }
  ]
}
```

**Resposta (207)**
```json
{
  "successes":[{"index":0,"appointmentId":"apt-1"}],
  "failures":[{"index":1,"error":{"code":409,"message":"Hour already booked"}}]
}
```

**Erros:** 207/201/409/422/404/401/403/500.

---

#### POST `/exclusions/bulk` — Criar exclusões em lote
**Body**
```json
{
  "items":[
    {
      "scheduleId":"sch-1",
      "type":"ExcludeRange",
      "title":"Almoço",
      "startTime":"12:00",
      "endTime":"13:00",
      "typeOfRecurrence":"DAILY"
    },
    {
      "scheduleId":"sch-2",
      "type":"ExcludeDay",
      "title":"Natal",
      "rrule":"FREQ=YEARLY;BYMONTH=12;BYMONTHDAY=25"
    }
  ]
}
```

**Resposta:** 207/201 conforme política.

**Erros:** 422 (combinação inválida), 404 (schedule), 409 (conflito), etc.

---

## 10. ADMINISTRAÇÃO

### 10.1. Gestão Administrativa — Conceitos
- Acesso restrito (role ADMIN/escopo elevado).
- Paginação e filtros avançados.
- Ações de manutenção potencialmente disruptivas: requerem `reason`, `requestedBy`, `confirmation=true`.

---

#### GET `/admin/schedules` — Todas as agendas (admin)
**Filtros:** healthUnitTaxId, specialityId, createdAtFrom/To, deleted=include|only|exclude, hasSlots, hasAppointments.

**Resposta (200)**
```json
{
  "data":[{"id":"sch-1","healthUnitTaxId":"540102938","specialityId":"CARDIOLOGY","deletedAt":null}],
  "pagination":{"page":1,"limit":20,"total":120}
}
```

**Erros:** 403/500.

---

#### GET `/admin/appointments` — Todas as marcações (admin)
**Filtros:** statusIn, professionalTaxId, patientTaxId, selectedHourFrom/To, healthUnitTaxId, specialityId.

**Resposta:** itens + paginação.

**Erros:** 403/500.

---

#### GET `/admin/slots` — Todos os slots (admin)
**Filtros:** scheduleId, dateFrom/To, isClosed, mode (derivado: TIMED/capacity), hasHours (true/false).

**Resposta:** itens + paginação.

**Erros:** 403/500.

---

#### POST `/admin/cleanup` — Limpeza de dados órfãos
**Objetivo:** identificar e (opcional) remover logicamente registos órfãos (Appointment sem Slot, Slot sem Schedule, etc.).

**Body**
```json
{
  "actions":["SCAN","SOFT_DELETE"], // SCAN apenas audita
  "entities":["appointments","slots","exclusions"],
  "dryRun": true,
  "reason": "Rotina mensal de manutenção",
  "requestedBy": "admin@hospital",
  "confirmation": true
}
```

**Resposta (200)**
```json
{
  "summary":{
    "appointments":{"orphansFound":3,"softDeleted":0},
    "slots":{"orphansFound":1,"softDeleted":1}
  },
  "dryRun": true
}
```

**Erros**

| Código | Quando |
|---|---|
| 400 | confirmation ausente em ações destrutivas |
| 403 | Sem perfil admin |
| 409 | Bloqueio de manutenção activo |
| 500 | Falha inesperada |

---

#### POST `/admin/recalculate-capacities` — Recalcular capacidades
**Objetivo:** reprocessar markingLimit/isClosed para slots (ex.: após ajuste de regras).

**Body**
```json
{
  "scheduleIds": ["sch-1","sch-2"],
  "dateFrom": "2025-02-01",
  "dateTo": "2025-02-28",
  "mode": "ASYNC",
  "reason": "Recalcular após mudança de duração",
  "requestedBy": "admin@hospital",
  "confirmation": true
}
```

**Resposta**
- `202 Accepted` (job assíncrono): `{ "jobId": "job-rc-123" }`
- `200 OK` (sync curto): resumo de reprocessamento.

**Erros**

| Código | Quando |
|---|---|
| 400 | Janela inválida / sem confirmation |
| 403 | Sem perfil admin |
| 409 | Job similar já em curso |
| 503 | Worker indisponível |
| 500 | Falha inesperada |

---

## Boas Práticas Transversais
- Observabilidade: incluir `correlationId` em todas as respostas; logs com endpoint, filters, tempos.  
- Idempotência: use `Idempotency-Key` em endpoints de lote e processos longos.  
- Segurança: mascarar dados sensíveis (paciente) em relatórios quando escopo não permite PHI.  
- Paginação estável: `sortBy` + `sortOrder` + `page/limit` ou cursor para grandes volumes.  
- Timeouts: relatórios pesados → `202 Accepted` + job e endpoint de `GET /jobs/{id}` (opcional).
