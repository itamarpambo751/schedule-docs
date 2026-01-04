# Robustez de Produção — Recomendações

A tua base (modelagem, auditoria, geração de slots, exclusões, shadows, etc.) já está sólida. Para ficar realmente **robusto de produção**, seguem recomendações práticas e acionáveis.

> **Padrões gerais**
> - Datas em **UTC** (timestamptz).
> - Idempotência em operações reexecutáveis.
> - Observabilidade com *correlationId* e métricas.

---

## 1) Confiabilidade & Integridade de Dados

- **Transações consistentes**: operações que criam/alteram `Schedules` + geração de `Slots` (em lote) devem rodar em transações ou jobs assíncronos com reprocessamento idempotente.
- **Idempotência**:
  - `Idempotency-Key` (UUID) em endpoints de lote, geração de slots, webhooks, notificações.
  - Persistir dedupe em DB (chave única por `(endpoint, idempotencyKey)`).
- **Outbox Pattern**: publicar eventos (`schedule.created`, `slots.generated`, `appointment.booked`) com confiabilidade.
  - Na mesma transação da escrita no Postgres, gravar evento numa `outbox_events`.
  - Um *dispatcher* assíncrono lê e publica no broker; marca como entregue.
- **Concorrência**:
  - **Optimistic concurrency**: `ETag/If-Match` em `PUT/PATCH` de recursos.
  - **Pessimista/Lock** quando necessário (reserva): `SELECT ... FOR UPDATE SKIP LOCKED` ou lock granular (ex.: chave `scheduleId+specificDate`).
- **Constraints úteis**:
  - `@@unique([scheduleId, specificDate])` em `Slots`.
  - `@@unique([slotId, selectedHour, professionalTaxId])` em `Appointments` (já tens).
  - *Checks* de coerência garantidos na aplicação + testes (ex.: `startTime < endTime`).

---

## 2) Escalabilidade & Performance

- **CQRS/OLAP para relatórios**:
  - Escrita: Postgres (modelo transacional).
  - Leitura analítica: read-model (ClickHouse/BigQuery/Elastic) ou *materialized views*.
- **Índices**: revisar com `EXPLAIN/ANALYZE` periodicamente; monitorar scans e *bloat*.
- **Particionamento** por data em `Slots`/`Appointments` se o volume crescer.
- **Caching**:
  - Aplicação: enums/shadows com TTL.
  - HTTP: `Cache-Control`/ETag para `GET /enums/*`, `/health`, etc.
- **Jobs assíncronos**:
  - Fila (Redis, SQS, RabbitMQ) com *retries* + backoff exponencial + DLQ.
  - *Sharding* por `healthUnitTaxId` ou `scheduleId` para paralelizar sem colisão.

---

## 3) Observabilidade & Operação

- **OpenTelemetry** (traces, métricas, logs) com propagação de `correlationId`.
- **SLIs/SLOs**:
  - Latência p95/p99 por endpoint.
  - Taxa de erro (4xx/5xx) por funcionalidade (bookings, slots-generation).
  - Duração média de job de geração de slots; throughput do dispatcher do outbox.
- **Alerting**: 5xx, fila a crescer, *retries* acima do normal, SLO degradado.
- **Runbooks**: procedimentos de incidentes — “overbooking”, “duplicidade de agenda”, “fila parada”.
- **Health**:
  - `/health` (liveness/readiness) com check de DB/Queue.
  - `/metrics` (Prometheus) com contadores e histogramas.

---

## 4) Segurança & Compliance (LGPD)

- **Autenticação/Autorização**:
  - JWT curto + refresh.
  - **RBAC/ABAC** por `healthUnitTaxId`, `specialityId`, *scopes* de leitura/escrita.
- **Criptografia**:
  - TLS em trânsito.
  - Em repouso: TDE ou colunas sensíveis com KMS/Vault.
  - Segredos em Vault/SM + rotação.
- **Privacidade**:
  - Minimizar PII em relatórios (mascarar `patientTaxId` sem escopo PHI).
  - **Data retention** e políticas de *soft-delete* + *purge*.
- **Auditoria**:
  - Log de mutações (Mongo/ES) com **quem**, **o quê**, **quando**, **antes/depois**, **correlationId**.
  - Integridade (hash/cadeia) e retenção.

---

## 5) Resiliência de Integrações (Shadows & Micros externos)

- **Camada Shadow** (já tens): `ProfessionalShadow`, `SpecialityShadow` com TTL/`expiresAt`.
- **Circuit Breaker** + **Retries com jitter** para micros remotos.
- **Fallback**: usar sombra “fresca o suficiente” quando upstream falha.
- **Bulk refresh** noturno + **on-demand refresh** com rate-limit.
- **Deduplicação** com `externalHash` (evita processar payloads idênticos).

---

## 6) Robustez do Motor de Slots/Agendamento

- **Dois modos**:
  - **Duração**: grelha de horas em `hours[]` (timeboxes).
  - **Capacidade**: `hours = []`, usar apenas `markingLimit`; `Appointment` referencia só `slotId`.
- **Exclusões**:
  - **Prioridade**: `ExcludeDay > ExcludeRange`.
  - Nunca apagar `BOOKED/HELD`; se necessário, marcar `BLOCKED` e iniciar fluxo de remarcação.
- **Anti-overbooking**:
  - Reserva com *optimistic hold* (status `HELD`, TTL ~5min).
  - Se alta concorrência, lock transacional na linha do `Slot`.
- **Idempotência na reserva**:
  - `clientRequestId` no `POST /appointments` para evitar duplicidade em reenvios do cliente.

---

## 7) Deploy & Mudanças

- **Migrações seguras (Expand → Migrate Data → Contract)**:
  - Adicionar colunas com *default*; backfill; só depois remover legado.
- **Canary/Blue-Green** + **Feature flags** para trocar comportamento (ex.: capacidade vs grelha) sem downtime.
- **Backups & DR**:
  - Backups diários + testes de restauração.
  - Réplicas de leitura e plano de **RPO/RTO**.

---

## 8) API & Contratos

- **OpenAPI/Swagger** versionado (v1, v2).
- **Contract testing** (Pact) com consumidores.
- **Paginação** consistente (`page/limit` ou cursor).
- **Erros padronizados**: códigos, `correlationId`, `traceId`.
- **Rate limiting** por IP/cliente/role; **quotas** para endpoints pesados.

---

## 9) Qualidade & Testes

- **Unitários**: regras de slots, exclusões, merges, reservas.
- **Carga**: geração de slots (3 meses, 500 agendas), reservas concorrentes.
- **Caos**: falhar DB/Queue por N segundos → sistema retoma e mantém idempotência.
- **E2E**: fluxo completo de marcação, cancelamento, reagendamento, encaminhamento.
- **Linters/Pre-commit**: schema, migrations, OpenAPI, RRULE validator.

---

## 10) Roadmap Técnico (próximos passos práticos)

1. **Outbox + Dispatcher** (schema + worker) para eventos confiáveis.
2. **Reserva com optimistic hold** (tabela/Redis com TTL) + endpoints.
3. **Sombras com TTL** e endpoint `/shadow/refresh` com rate-limit.
4. **Modelo de relatórios**: materialized views ou read-model (ClickHouse).
5. **OpenTelemetry** + dashboards (latência, erros, jobs, occupancy) + alertas.
6. **Feature flags** para alternar “capacidade vs grelha” por agenda.
7. **Rate limit** e `202 Accepted` + jobs para operações longas.
