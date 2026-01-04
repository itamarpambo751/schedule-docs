# Auditoria de Mutações em MongoDB (com Outbox/CDC)

> Proposta de arquitectura para registar **actividades de mutação** (quem mudou o quê, quando e porquê) usando **MongoDB** como _data store_ de auditoria, mantendo o PostgreSQL transaccional limpo e consistente.

---

## 1. Porquê Mongo para Auditoria?

**Vantagens**
- **Volume alto / eventos verbosos**: armazena *before/after*, *diffs*, contexto de request, sem “inchar” o OLTP.
- **Consultas exploratórias**: dashboards, buscas textuais e filtros ad-hoc funcionam muito bem em modelo documento.
- **Retenção e particionamento** independentes do transaccional.
- **Esquema evolutivo**: adicionar campos novos nos eventos sem migrações rígidas.

**Riscos & Mitigações**
- **Dupla escrita inconsistente** (PG e Mongo): usar **Transactional Outbox** (preferido) ou **CDC** (Debezium).
- **Falhas parciais / retries**: chave de **idempotência** por evento + **DLQ** (dead-letter queue).
- **LGPD/Privacidade**: minimizar PII, mascarar campos sensíveis, definir retenção com **TTL**.
- **Desalinhamento temporal**: timestamps de servidor e usar `createdAt` do outbox como _source of truth_.

---

## 2. Padrões de Arquitectura

### 2.1 Transactional Outbox (Recomendado)
1. Na **mesma transacção** do `INSERT/UPDATE` do PostgreSQL, gravar uma linha em `audit_outbox` (append-only).
2. Um **worker** lê o outbox e **materializa** o evento de auditoria no Mongo (marca linha como processada).
3. **Garantias**: se a transacção do PG confirmou, o evento existe; consumo **at-least-once** com chaves idempotentes.

### 2.2 CDC (Change Data Capture)
- Usar **Debezium** para ler o **WAL** do PostgreSQL e publicar eventos (Kafka). Um *sink* grava no Mongo.
- Poupa alterações na app, mas requer infra de streaming.

> **Evitar** escrever diretamente no Mongo dentro da request **sem** outbox/CDC: risco de perder eventos em falhas parciais.

---

## 3. Esquema de Evento (Mongo)

**Campos mínimos recomendados**
- `eventId` (UUID), `occurredAt` (UTC), `recordedAt` (UTC do consumidor).
- **Quem**: `actor.type` (USER/SYSTEM), `actor.id`, `impersonatedBy?`, `roles`, `ip`, `userAgent`.
- **O quê**: `entity.type` (Schedules/Slots/Appointment/…), `entity.id`.
- **Operação**: `op` (CREATE/UPDATE/DELETE/SOFT_DELETE/RESTORE).
- **Onde**: `healthUnitTaxId?`, `specialityId?` (quando aplicável).
- **Porquê**: `reason` / `comment`.
- **Correlação**: `requestId`, `correlationId`, `sourceSystem`.
- **Estado**: `before`, `after` e/ou **delta/diff** (RFC 6902 opcional).
- **Idempotência**: `idempotencyKey` ou `externalHash`.

**Exemplo**
```json
{
  "eventId": "7b2f2c1e-3e57-4c5c-9e7d-1d1bfa3e9a9a",
  "occurredAt": "2025-01-25T10:15:33.421Z",
  "recordedAt": "2025-01-25T10:15:33.900Z",
  "sourceSystem": "scheduling-api",
  "requestId": "req_3f9a",
  "correlationId": "corr_12ab",
  "op": "UPDATE",
  "entity": { "type": "Schedules", "id": "sch_123" },
  "scope": { "healthUnitTaxId": "540102938", "specialityId": "CARDIOLOGY" },
  "actor": {
    "type": "USER",
    "id": "admin@hospital",
    "roles": ["SCHED_ADMIN"],
    "ip": "203.0.113.10",
    "userAgent": "web/1.2.3"
  },
  "diff": {
    "set": { "appointmentDurationInMinutes": 20 },
    "unset": [],
    "replace": []
  },
  "before": { "appointmentDurationInMinutes": 30 },
  "after":  { "appointmentDurationInMinutes": 20 },
  "tags": ["policy-change","high-impact"],
  "idempotencyKey": "idempo-abc123"
}
```

**Índices úteis no Mongo**
- `entity.type + entity.id + occurredAt` (timeline por registo).
- `actor.id + occurredAt` (auditoria por utilizador).
- `op + occurredAt` (relatórios de operações).
- `sourceSystem + requestId` (debug de uma chamada).
- `occurredAt` com **coleções por mês** ou **time-series**.

---

## 4. Implementação com Prisma (Transactional Outbox)

### 4.1 Tabela Outbox (PostgreSQL)
```sql
CREATE TABLE audit_outbox (
  id UUID PRIMARY KEY,
  occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  entity_type TEXT NOT NULL,
  entity_id TEXT NOT NULL,
  op TEXT NOT NULL, -- CREATE/UPDATE/DELETE/SOFT_DELETE/RESTORE
  actor JSONB NOT NULL, -- type/id/roles/ip/ua
  scope JSONB,
  diff JSONB,
  before JSONB,
  after JSONB,
  request_id TEXT,
  correlation_id TEXT,
  source_system TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'PENDING', -- PENDING/PROCESSED/FAILED
  attempts INT NOT NULL DEFAULT 0
);
CREATE INDEX ON audit_outbox (status, occurred_at);
```

### 4.2 Middleware Prisma (esboço)
- Interceptar `create/update/delete` e montar **evento outbox** na **mesma transacção**.
- Incluir `before/after` e `diff` (ou apenas campos alterados).

Pseudo-código:
```ts
prisma.$use(async (params, next) => {
  const resultBefore = await snapshotBeforeIfNeeded(params);
  const result = await next(params);
  const resultAfter = await snapshotAfterIfNeeded(params, result);

  const outboxEvent = buildOutboxEvent({
    params, resultBefore, resultAfter,
    actor: getActorFromContext(), // userId/roles/ip/ua
    requestId: ctx.requestId, correlationId: ctx.correlationId
  });

  await prisma.audit_outbox.create({ data: outboxEvent }); // dentro da mesma TX
  return result;
});
```

### 4.3 Worker (consumer)
- Lê `audit_outbox` em batches (`status = PENDING`), grava documento no Mongo e marca como `PROCESSED`.
- **Idempotência**: usar `eventId` como `_id` no Mongo; `upsert` evita duplicados.
- **DLQ**: em falha persistente, mover para `FAILED` com `attempts++` e registar o erro.

---

## 5. Políticas de Privacidade e Retenção (LGPD)

- **Minimização**: guardar *apenas* o necessário para auditoria e troubleshooting.
- **Mascaramento**: ofuscar PII (ex.: `patientTaxId`) quando o escopo do actor não permite ver PHI.
- **TTL**: coleções com TTL (p.ex., 12–24 meses) ou *sharding/particionamento* por mês.
- **Direito ao esquecimento**: suportar anonimização dos campos pessoais em eventos históricos, se exigido.
- **Acesso**: RBAC/ABAC também no data lake de auditoria (dashboards com escopos).

---

## 6. Observabilidade e Operação

- **Correlation IDs** em todos os endpoints e eventos.
- **Métricas**: taxa de processamento do outbox, lag, DLQ size, erros por tipo de operação.
- **Alertas**: atraso do consumidor, explosão de `FAILED`, acumulação de `PENDING`.
- **Backfills**: tarefas para reprocessar janelas históricas (com filtros por data/status).

---

## 7. Alternativa: Auditoria no PostgreSQL

Quando consistência transaccional estrita é prioridade: manter uma **tabela append-only** no próprio PG, com **particionamento** (por mês) e **índices** adequados.  
Prós: simplicidade e ACID. Contras: base OLTP cresce e relatórios pesados competem com workload transaccional.

---

## 8. Recomendações Finais

- **Sim, usar Mongo para auditoria** é ótimo para flexibilidade analítica e payloads ricos.
- **Outbox/CDC** é **obrigatório** para consistência e resiliência.
- Definir **esquema mínimo** de evento, **índices**, **retenção** e **políticas de privacidade**.
- Separar **eventos de negócio** (`appointment.rescheduled`) de **logs de auditoria** (mutações campo-a-campo) — ambos úteis e complementares.

---
