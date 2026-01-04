# 8. VALIDAÇÕES E UTILITÁRIOS — ESPECIFICAÇÃO DE ENDPOINTS
> Todas as datas em **UTC** (ISO 8601). Autenticação via **Bearer Token**. Autorizações via **RBAC/ABAC**.

---

## 8.1. Validações

### POST `/validate/schedule` — Validar configuração de agenda
**Objetivo:** verificar se um payload de `Schedule` é coerente e aceitável antes de criar/atualizar.

**Validações executadas (exemplos):**
- **Campos obrigatórios e formatos:** `healthUnitTaxId`, `specialityId`, `typeOfService`, `typeOfSchedule` (enums válidos).
- **Coerência temporal:** `startTime < endTime`, `startDate ≤ endDate` (quando presente).
- **Modo de capacidade/tempo:** obrigatório fornecer **ou** `appointmentTIMEDInMinutes > 0` **ou** `maxAppointmentsPerSlot ≥ 1`.  
  - Se ambos vierem, o **modo tempo** (TIMED) **prevalece**.
- **Recorrência:** se `typeOfRecurrence ≠ NONE` ⇒ `weekDays` **não vazio**.
- **Profissionais:** quando `availableProfessionalTaxIds` vier preenchido, validar
  - existência dos `ProfessionalShadow`/`Professional`,
  - **compatibilidade com `ProfessionalWorkTime`** (mesmo `healthUnitTaxId`, `weekDay`, faixa `startAt..endsAt` cobre a janela diária),
  - especialidade compatível (`specialityId` do `Schedule` ∈ especialidades do profissional naquele *hospital*).
- **Exclusões aplicáveis:** se houver `ExcludeRange`/`ExcludeDay` activos que **anulem integralmente** a janela diária, retornar *warning* (ou *error* se a política local exigir).
- **Políticas locais (opcional):** conflito lógico com outra agenda activa (unicidade por `healthUnitTaxId + specialityId + typeOfService + typeOfSchedule + typeOfRecurrence + weekDays`).

**Request (exemplo):**
```json
{
  "healthUnitTaxId": "540102938",
  "specialityId": "CARDIOLOGY",
  "weekDays": ["MONDAY", "WEDNESDAY"],
  "startTime": "2025-11-01T08:00:00Z",
  "endTime": "2025-11-01T16:00:00Z",
  "appointmentTIMEDInMinutes": 30,
  "typeOfRecurrence": "WEEKLY",
  "startDate": "2025-11-01T00:00:00Z",
  "endDate": "2026-01-31T00:00:00Z",
  "typeOfSchedule": "CONSULTATION",
  "typeOfService": "IN_PERSON",
  "advanceCancellationInHours": 12,
  "deadlineForSlotBookingInHours": 2,
  "availableProfessionalTaxIds": ["BI12345", "BI67890"],
  "automaticallyGenerateSlots": true
}
```

**Response (200 OK):**
```json
{
  "valid": true,
  "warnings": [
    {"code":"WINDOW_NOT_MULTIPLE_OF_STEP","message":"A janela diária não é múltipla do step (30min). Últimos minutos serão ignorados."},
    {"code":"EXCLUDES_OVERLAP","message":"Há exclusões que reduzem a disponibilidade útil em 45%."}
  ],
  "effectiveRules": {
    "recurrence": "WEEKLY",
    "weekDays": ["MONDAY","WEDNESDAY"],
    "mode": "TIMED"
  }
}
```

**Erros comuns:**
- **422 Unprocessable Entity** — regra de negócio inválida (ex.: `weekDays` vazio com `typeOfRecurrence ≠ NONE`; profissional fora do turno).
- **409 Conflict** — sobreposição com agenda existente (quando política estiver activa).
- **400 Bad Request** — payload mal formatado.
- **401/403** — sem autenticação/permissão.

---

### POST `/validate/slot-availability` — Validar disponibilidade
**Objetivo:** confirmar se um `Slot` (por `slotId` + opcional `selectedHour`) tem capacidade/disponibilidade para criar um `Appointment`.

**Regras:**
- **Modo duração (com horas):** requer `slotId` + `selectedHour`.
  - Verifica se `selectedHour` existe em `slots.hours[]` com `status = "AVAILABLE"`,
  - se o `professionalTaxId` é permitido (herdado do `Schedule` ou definido no `Slot`),
  - se o `selectedHour` **não coincide** com `ExcludeRange`/`ExcludeDay` **publicados**.
- **Modo capacidade (sem horas):** requer apenas `slotId`.
  - Verifica `countFilled < markingLimit` e `isClosed = false`.

**Request (ex.: modo duração):**
```json
{
  "slotId": "slot-123",
  "selectedHour": "2025-12-14T14:00:00Z",
  "professionalTaxId": "BI12345",
  "patientTaxId": "BI99999"
}
```

**Response (200 OK):**
```json
{
  "available": true,
  "mode": "TIMED",
  "reason": null
}
```

**Erros comuns:**
- **404 Not Found** — `slotId` inexistente.
- **409 Conflict** — hora já ocupada/held (BOOKED/HELD/BLOCKED) ou capacidade esgotada.
- **422 Unprocessable Entity** — `selectedHour` obrigatório no modo duração; `professionalTaxId` não autorizado no slot.
- **403 Forbidden** — utilizador sem permissão para reservar nesse `Schedule/healthUnit`.

---

### POST `/validate/slot-publication` — Validar publicação de slots
**Objetivo:** antes de **publicar** slots (workflow: *DRAFT → READY_FOR_REVIEW → PUBLISHED*), garantir que não existem conflitos.

**Checks executados:**
- Slot **não deletado** e **não fechado** (`isClosed=false`).
- `hours[]` (quando modo duração) **válidos** e coerentes com `appointmentTIMEDInMinutes` do `Schedule`.
- Sem colisão com `ExcludeRange/ExcludeDay` **activos** no intervalo do slot.
- Profissionais válidos (herdados ou específicos) **com turnos em `ProfessionalWorkTime`**.
- **Capacidade** (`markingLimit`) ≥ número de `hours` **AVAILABLE** (quando aplicável).
- Estado do `Schedule` **activo** no período (`startDate..endDate`).

**Response (200 OK):**
```json
{ "publishable": true, "warnings": [] }
```

**Erros comuns:** 409 (conflito de exclusões), 422 (horas inválidas), 404 (schedule/slot inexistente).

---

### POST `/validate/appointment` — Validar marcação
**Objetivo:** validar a criação de um `Appointment` antes do commit.

**Validações:**
- Consistência com o `Schedule/Slot` alvo (modo duração vs capacidade).
- Restrições de prazo: `deadlineForSlotBookingInHours`, janelas mín./máx. desde *agora*.
- Políticas de cancelamento/remarking (quando update).
- Conflitos com outros appointments do **mesmo profissional**/**mesmo paciente** para a mesma hora (índices compostos).
- Campos obrigatórios: `patientTaxId`, `professionalTaxId`, `typeOfService` (telemedicina exige `roomLink` na confirmação).

**Request (ex.):**
```json
{
  "slotId": "slot-123",
  "selectedHour": "2025-12-14T14:00:00Z",
  "patientTaxId": "BI99999",
  "professionalTaxId": "BI12345",
  "typeOfService": "IN_PERSON"
}
```

**Response (200 OK):**
```json
{
  "valid": true,
  "normalized": {
    "slotId": "slot-123",
    "selectedHour": "2025-12-14T14:00:00Z",
    "status": "PENDING"
  }
}
```

**Erros comuns:**
- **422** — viola regra (deadline, hora inexistente/indisponível no slot, enum inválido).
- **409** — colisão com outra marcação do mesmo prof./paciente na mesma hora.
- **404** — slot/schedule inexistente.
- **403** — ACL.

---

### POST `/validate/rrule` — Validar regra RRULE
**Objetivo:** verificar sintaxe e expansão de uma **RRULE (RFC 5545)** e opcionalmente listar ocorrências numa janela.

**Request:**
```json
{
  "rrule": "FREQ=WEEKLY;INTERVAL=2;BYDAY=SA",
  "from": "2025-01-01T00:00:00Z",
  "to": "2025-03-01T00:00:00Z",
  "tz": "UTC",
  "maxInstances": 100
}
```

**Processamento:**
- Valida sintaxe (`FREQ`, `INTERVAL`, `BYDAY`, etc.).
- Expande ocorrências entre `from` e `to` (limitar por `maxInstances`).
- Opcional: valida compatibilidade com `weekDays`/`typeOfRecurrence` se for aplicado em `ExcludeRange/ExcludeDay`.

**Response (200 OK):**
```json
{
  "valid": true,
  "occurrences": [
    "2025-01-04T00:00:00Z",
    "2025-01-18T00:00:00Z",
    "2025-02-01T00:00:00Z",
    "2025-02-15T00:00:00Z"
  ],
  "truncated": false
}
```

**Erros comuns:** **422** (RRULE inválida ou janela `from > to`), **400** (campos em falta), **413** (instâncias demais sem `maxInstances`).

---

## 8.2. Utilidades (Enums)
Todos devolvem listas estáticas derivadas dos **enums** do schema (úteis para UI/validação).  
**Resposta padrão:** `{"values": ["ENUM1","ENUM2", ...]}`

- `GET /enums/type-of-recurrence`
  ```json
  ["NONE","DAILY","WEEKLY","MONTHLY","QUARTERLY","SEMIANNUALLY","ANNUALLY","CUSTOM"]
  ```
- `GET /enums/type-of-schedule`
  ```json
  ["CONSULTATION","VACCINE","EXAM","RETURN"]
  ```
- `GET /enums/type-of-service`
  ```json
  ["IN_PERSON","TELEMEDICINE","HOME"]
  ```
- `GET /enums/week-days`
  ```json
  ["SUNDAY","MONDAY","TUESDAY","WEDNESDAY","THURSDAY","FRIDAY","SATURDAY"]
  ```
- `GET /enums/appointment-status`
  ```json
  ["PENDING","CONFIRMED","CHECKED_IN","IN_SESSION","RESCHEDULED","COMPLETED","FOLLOW_UP_REQUIRED","DOCUMENT_PENDING","CANCELLED_BY_PATIENT","CANCELLED_BY_PROFESSIONAL","CANCELLED_BY_SYSTEM","NO_SHOW","EXPIRED"]
  ```
- `GET /enums/forwarding-status`
  ```json
  ["PENDING","SENT","RECEIVED","ACCEPTED","REJECTED","COMPLETED","CANCELLED"]
  ```
- `GET /enums/priority`
  ```json
  ["NORMAL","URGENT","SERIOUS"]
  ```

**Boas práticas de caching:** `ETag`, `Cache-Control: max-age=3600`, *invalidação* ao publicar nova versão do schema.

---

## 8.3. Health & Readiness

### GET `/health` — Liveness Probe
**Objetivo:** indicar saúde do processo para load balancers/monitores.

**Checks típicos:**
- Conectividade DB (*ping*/consulta leve).
- Fila de jobs (opcional): lag/consumidores activos.
- Versão/commit e tempo de arranque.

**Response (200 OK):**
```json
{
  "status": "ok",
  "uptimeSec": 123456,
  "version": "1.3.7",
  "commit": "abc1234",
  "checks": { "db": "ok", "queue": "ok" },
  "timestamp": "2025-01-05T10:00:00Z"
}
```

**Erros:**
- **503 Service Unavailable** — check crítico falhou (ex.: DB down).
- **200 OK** com `"status": "degraded"` — dependências não‑críticas falharam.

### GET `/ready` — Readiness Probe
**Objetivo:** confirmar que **todas as dependências** necessárias para servir tráfego estão OK.
- Verifica **migrates** aplicadas, conexões a **todos** os brokers/DBs e acesso a caches necessários.

**Response (200 OK):**
```json
{ "status": "ready", "checks": { "db": "ok", "broker": "ok", "migrations": "ok" } }
```

**Erros:** **503** quando algum requisito não está pronto.

---

### GET `/metrics` — Métricas do sistema
**Objetivo:** expor métricas para Prometheus/Grafana ou painel interno.

**Conteúdos típicos:**
- Contadores/histogramas: **requisições**, **latência** por endpoint, **erros** por código.
- Métricas de domínio: `schedules_created_total`, `slots_generated_total`, `appointments_booked_total`, `forwardings_pending_total`, `reschedules_pending_total`.
- Uso de recursos (opcional): memória/CPU.

**Formato:**
- **Prometheus text** por omissão (`Content-Type: text/plain; version=0.0.4`).
- **JSON** opcional via `Accept: application/json`.

**Exemplo (Prometheus-like):**
```
# HELP http_requests_total Total de requests HTTP.
# TYPE http_requests_total counter
http_requests_total{method="POST",endpoint="/validate/schedule",code="200"} 153

# HELP appointments_booked_total Marcações confirmadas
# TYPE appointments_booked_total counter
appointments_booked_total 892
```

**Erros:**
- **503** — exportador indisponível.
- **200 OK** com subset de métricas quando algumas fontes falham.

---

## 8.4. Observabilidade, Segurança & Notas Finais
- **Idempotência:** validadores não alteram estado; apenas calculam/retornam resultados.
- **Segurança:** validar **ACL** e **escopos** nos validadores que expõem informação sensível (ex.: disponibilidade real).
- **Audit Trail:** incluir `correlationId` nas respostas; logar inputs normalizados, tempo de execução e decisão (valid/warnings/errors).
- **Consistência:** todas as datas em **UTC**; enums exactamente como no schema.
- **Workflow de Publicação de Slots:** usar estados `DRAFT → READY_FOR_REVIEW → PUBLISHED`; o endpoint `/validate/slot-publication` deve ser executado **antes** do *publish*.
- **Integração com Sombra (Shadow):** quando aplicável, resolver `specialities`/profissionais via tabelas *Shadow* e cruzar com `ProfessionalWorkTime` por `healthUnitTaxId`.
