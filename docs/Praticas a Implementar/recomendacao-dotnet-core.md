# 🧠 Recomendação: .NET para o Core (CPU/Concorrência Pesada)

## 🔍 Por que .NET no teu caso

O **.NET** é altamente indicado para workloads com **processamento intensivo e paralelismo real**, oferecendo previsibilidade, escalabilidade e ferramentas maduras de observabilidade e resiliência.

### 💪 Motivos principais

- **CPU/Concorrência real**:  
  `ThreadPool`, `Parallel.*`, `System.Threading.Channels`, `IAsyncEnumerable`, `Span<T>`, SIMD e `BackgroundService` permitem **paralelismo previsível** e **alto throughput**.
- **Workers & schedulers robustos**:  
  `BackgroundService` + `Quartz.NET` ou `Hangfire` garantem execução confiável de **jobs agendados** (ex.: geração de slots, reconciliação, relatórios).
- **Contratos sólidos**:  
  gRPC para microserviços e REST para clientes, com **tipagem forte** e **tooling avançado**.
- **Observabilidade “bater-e-andar”**:  
  `OpenTelemetry`, `HealthChecks` e `EventCounters` oferecem **visibilidade nativa** de performance e saúde dos serviços.

---

## 🏗️ Arquitetura Proposta (High-Level)

### 🧩 Serviços

#### **scheduling-core** (ASP.NET Core)
- Exposição **REST/gRPC**: CRUD de `Schedules`, `Slots`, `Appointments` e `Exclusions`.  
- Relatórios leves e síncronos (para dashboards).  
- Health endpoints e métricas.

#### **slot-worker** (Worker Service .NET)
- Geração e *merge* de slots por janelas grandes (como definido na lógica de negócios).  
- Aplicação de `ExcludeDay` / `ExcludeRange` e **reconciliação**.  
- **Consumidor de fila** (Kafka, RabbitMQ ou Redis Streams).  
- Execução de tarefas recorrentes como “recalculate capacities” e “cleanup”.

#### **(Opcional) analytics-agg** (Worker)
- ETL e agregação de dados: ocupação, *no-show*, cancelamentos.  
- Geração de *materialized views* para relatórios.

---

## ⚙️ Infraestrutura

| Componente | Tecnologia | Finalidade |
|-------------|-------------|-------------|
| **Banco de Dados** | PostgreSQL + EF Core + Dapper | Persistência e leitura otimizada |
| **Cache** | Redis | Locks, dedupe, throttling e cache de lookups |
| **Fila** | Kafka ou RabbitMQ | Jobs assíncronos e geração de relatórios |
| **Logs & Métricas** | OpenTelemetry → Tempo/Prometheus + Serilog | Observabilidade e logs estruturados |

---

## 🧱 Stack Técnica Sugerida (.NET 8)

- **Web/API** → ASP.NET Core Minimal APIs + gRPC  
- **ORM** → EF Core (Npgsql) + Dapper (para leituras pesadas)  
- **Jobs** → BackgroundService + Quartz.NET (ou Hangfire se quiser UI)  
- **Resiliência** → Polly (retry, circuit-breaker, timeout, bulkhead)  
- **Observabilidade** → OpenTelemetry (Tracing/Metrics/Logs) + HealthChecks  
- **Segurança** → JWT Bearer + RBAC/ABAC  
- **Validação** → FluentValidation  
- **Mapeamento** → Mapster (ou AutoMapper)  
- **Docs** → Swashbuckle (OpenAPI/Swagger)  
- **Testes** → xUnit, Respawn, Testcontainers (Postgres/Redis/Kafka em CI)

---

## ⚡ Padrões de Concorrência (onde .NET brilha)

### 🧵 Pipelines de geração com Channels

Fluxo:
```
producer (expande recorrência) → channel → consumers (N paralelos)
```

Cada consumidor:
- Calcula grelha/capacidade.  
- Aplica exclusões.  
- Executa **upsert idempotente** de Slots.

### 🔒 Locks leves

Uso de **Redis RedLock** por janela `(scheduleId, from, to)` para evitar jobs duplicados.

### 🔁 I/O bound

Uso de `IAsyncEnumerable<T>` para **streaming eficiente** em leitura e escrita *chunked*.

---

## 🗂️ Layout de Solução (Exemplo)

```
/src
  Scheduling.Api           // ASP.NET Core Minimal APIs + gRPC
  Scheduling.Domain        // Entidades e regras de negócio puras
  Scheduling.Application   // Use-cases, Services, Validators
  Scheduling.Infrastructure// EF Core, Dapper, Mensageria, Redis, OTel
  Scheduling.Worker        // BackgroundService: geração de slots, relatórios, reconciliação

/tests
  Scheduling.UnitTests
  Scheduling.IntegrationTests
  Scheduling.LoadTests     // opcional (NBomber/k6)
```

---

## 🗓️ Roadmap de Implementação (2–3 Semanas)

### **Semana 1**
- Bootstrapping do repositório.  
- `docker-compose` com Postgres, Redis e Kafka.  
- Modelagem EF Core + migrações.  
- CRUD Minimal APIs + validação FluentValidation.  
- HealthChecks + OpenTelemetry (traços e métricas básicas).

### **Semana 2**
- Implementar **Scheduling.Worker** com Channels.  
- Geração idempotente de slots (capacidade + duração) com locks Redis.  
- Reconciliação “on-demand” via endpoint `/schedules/{id}/generate-slots`.  
- Sincronização de tabelas sombreadas (shadow tables).

### **Semana 3**
- Relatórios agregados (utilização/ocupação).  
- Harden de resiliência (Polly policies).  
- Testes de carga (k6/NBomber) e tunning (pool size, batch size).

---

## ⚙️ Configurações Críticas de Produção

| Componente | Configuração |
|-------------|--------------|
| **Kestrel** | HTTP/2 (gRPC), ajuste de ThreadPool min/max, GC Server |
| **EF Core** | `AsNoTracking`, `BatchSize` otimizado, keepalive ativo |
| **Npgsql** | Pool Size calibrado, multiplexing ON |
| **Redis** | Timeouts curtos + retries com jitter (Polly) |
| **Kafka/Rabbit** | Consumer groups e backpressure ajustado |
| **OpenTelemetry** | Amostragem 1–5% (ajustável conforme incidentes) |

---

## 🔗 Integração com Front e Microserviços

- **REST** → consumo externo (front, parceiros).  
- **gRPC** → comunicação interna (baixa latência).  
- DTOs versionados (`v1`, `v2`) com *feature flags* para migrações seguras.

---

## ⚠️ Plano de Risco e Mitigação

| Risco | Mitigação |
|--------|------------|
| **Sobrecarga de CPU** | `bulkhead` + fila com backpressure |
| **Jobs duplicados** | Idempotência + Redis/Kafka locks |
| **Migrações grandes** | Blue/Green deploy + feature toggles + expand/contract |
| **DST/Fusos** | UTC end-to-end + testes de fronteira mês/ano |

---

## 🌐 Quando (e como) adicionar Node depois

Se precisar de **BFF**, **WebSockets**, **GraphQL** ou **edge caching**, adicione um **API Gateway em Node (Fastify/Nest)** na frente:

- Traduz REST/gRPC, agrega respostas e faz *schema stitching* para o front.  
- Mantém o **core em .NET**, onde está o trabalho pesado.

---

## ✅ Conclusão

O .NET 8 fornece uma base altamente eficiente e estável para o **motor de agendamento, geração de slots e reconciliação**, com:
- Concorrência previsível e segura,  
- Ferramentas nativas para resiliência e observabilidade,  
- E uma arquitetura modular que permite expandir com Node.js ou outros serviços no futuro.
