
# Shadow (SpecialityShadow) — Cache Persistente de Especialidades

> **Resumo:** O *shadow* é uma tabela local (na tua BD) gerida pelo Prisma que guarda um **espelho mínimo** dos dados de um micro-serviço externo (p.ex., **Specialities**). Serve para **validação**, **resiliência** e **reporting** sem depender 100% do micro remoto. É **read-only** para o teu domínio (só o processo de sync escreve).

---

## 1) Porquê usar Shadow?

- **Resiliência:** permite continuar a validar `specialityId` quando o micro externo está lento ou indisponível.
- **Desacoplamento:** remove FKs fortes para domínios externos; o teu schema continua independente.
- **Performance:** menos *round-trips* em operações frequentes (criação de agendas, turnos, relatórios).
- **Reporting:** nomes/descrições human-friendly sem ir ao micro; análises históricas consistentes.

> Padrão recomendado: **ID-only em modelos de negócio** (ex.: `Schedules.specialityId`) + **tabela Shadow** com os metadados mínimos.

---

## 2) Modelo Prisma (exemplo)

```prisma
model SpecialityShadow {
  id        String   @id                  // ID do micro externo
  name      String
  updatedAt DateTime @updatedAt @db.Timestamptz
  fetchedAt DateTime @default(now()) @db.Timestamptz

  // (Opcional) metadados de origem
  // sourceEtag        String?
  // sourceUpdatedAt   DateTime? @db.Timestamptz
}
```

- **É uma tabela normal da BD**: faz parte das migrations, CRUD via Prisma Client e pode ter *indexes*.
- **Sem FKs** para os teus modelos de negócio (evita acoplamento).

---

## 3) Onde o Shadow é usado no fluxo

### Escrita (Create/Update)
1. Recebes payload (ex.: criar `Schedules` com `specialityId`).
2. **Valida `specialityId` via Shadow**:
   - Se existe → prossegue.
   - Se **não** existe → chama o micro externo:
     - **200** → faz **upsert** no shadow e continua.
     - **404** → responde **422 Unprocessable Entity** (specialityId inválido).
     - **5xx/timeout** → responde **503 Service Unavailable** (ou modo degradado, ver abaixo).
3. Persiste o teu modelo (`Schedules`, `ProfessionalWorkTime`, etc.).

### Leitura (listas/relatórios)
- Faz *join lógico* com o Shadow (por `id`) para trazer `name`/metadados sem ir ao micro.

---

## 4) Padrões de sincronização (escolhe 1 ou combina)

1. **Pull periódico** (cron/worker)
   - De X em X minutos, `GET /specialities` do micro → `upsert` em `SpecialityShadow`.
2. **On-demand (write-through)**
   - Na primeira utilização de um `specialityId` desconhecido: consulta o micro e persiste no shadow.
3. **Eventos/Webhooks** (ideal se existir)
   - Micro de Especialidades emite `speciality.updated`; o teu serviço processa e actualiza o shadow.

> **Recomendação prática:** **On-demand + Pull periódico**. Assim reduzes *misses* e manténs dados frescos.

---

## 5) Política de validade (TTL) e “modo degradado”

- Mantém um **TTL lógico** (p.ex., 24h). Se `fetchedAt` ultrapassar o TTL, tenta refrescar antes de servir.
- Se o micro estiver indisponível **e já tens o shadow**:
  - **Modo degradado (opcional):** servir a partir do cache **com aviso** (ex.: header `X-Degraded: 1`).
  - Ou **falhar** com **503** conforme a política do produto.
- Guarda **logs/metrics**: contagem de *cache hit*, *miss*, falhas de sync, latências.

---

## 6) Tratamento de erros (recomendações)

| Situação | Código | Mensagem |
|---|---|---|
| `specialityId` não existe no micro | **422** | `Invalid specialityId` |
| Micro de especialidades indisponível | **503** | `Speciality service unavailable` |
| Timeout ao contactar o micro | **504** | `Upstream timeout on speciality service` |
| Falha inesperada no sync | **500** | `Shadow sync failed` |

> Distinguir **validade de dados (422)** de **indisponibilidade externa (503/504)** dá UX melhor e facilita *retry/backoff* no cliente.

---

## 7) Exemplos (Node/TypeScript + Prisma)

### 7.1. Validação “ID-only + shadow”

```ts
import { prisma } from './prisma';

type RemoteSpeciality = { id: string; name: string };

async function ensureSpeciality(id: string, opts: {
  fetchRemote: (id: string) => Promise<RemoteSpeciality | null>,
  allowDegradedWithCache?: boolean
}) {
  // 1) Cache local
  const cached = await prisma.specialityShadow.findUnique({ where: { id } });
  if (cached) return cached;

  // 2) Consulta micro (on-demand)
  try {
    const remote = await opts.fetchRemote(id);
    if (!remote) {
      const err: any = new Error('Invalid specialityId');
      err.status = 422;
      throw err;
    }
    // 3) Upsert no shadow
    await prisma.specialityShadow.upsert({
      where: { id: remote.id },
      update: { name: remote.name },
      create: { id: remote.id, name: remote.name }
    });
    return remote;
  } catch (e: any) {
    if (opts.allowDegradedWithCache && cached) return cached;
    if (e.status === 422) throw e; // inválido
    const err: any = new Error('Speciality service unavailable');
    err.status = 503;
    throw err;
  }
}
```

### 7.2. Uso no serviço de criação de `Schedules`

```ts
async function createSchedule(input: {
  healthUnitTaxId: string;
  specialityId: string;
  // ...
}) {
  await ensureSpeciality(input.specialityId, {
    fetchRemote: getSpecialityFromMicro, // a tua função HTTP
    allowDegradedWithCache: false
  });

  return prisma.schedules.create({ data: { ...input, createdBy: '...' } });
}
```

### 7.3. Job de refresh periódico (cron/worker)

```ts
async function refreshSpecialitiesFromMicro() {
  const list = await fetchAllFromMicro(); // faz paginação se necessário
  await prisma.$transaction(
    list.map(s =>
      prisma.specialityShadow.upsert({
        where: { id: s.id },
        update: { name: s.name },
        create: { id: s.id, name: s.name }
      })
    ),
    { maxWait: 10_000, timeout: 60_000 }
  );
}
```

---

## 8) Segurança, Privacidade e Auditoria

- **Dados mínimos** no Shadow (evita PII/PHI desnecessária). Geralmente, `id` e `name` bastam.
- **RBAC/ABAC**: endpoints de sync só para perfis internos/ops.
- **Auditoria**: regista *who/when* actualizou (logs do worker), `correlationId`, sucesso/falha.
- **Rate limiting/backoff** ao chamar o micro externo.
- **Circuit breaker** em caso de indisponibilidade prolongada.

---

## 9) Migração e primeiros passos

1. **Criar a tabela** `SpecialityShadow` via Prisma Migration.
2. **Implementar** `ensureSpeciality()` e usar em todos os *writes* que consomem `specialityId`.
3. **Adicionar** job de **refresh** (cron) + métricas/alertas.
4. **Documentar** política de TTL, modo degradado e mapeamento de erros (422 vs 503/504).
5. **Opcional:** endpoint `/admin/speciality-shadow/sync` para sync manual com `Idempotency-Key`.

---

## 10) FAQ rápida

- **“O Shadow cria FK com `Schedules`?”**  
  Não. Mantém-se **desacoplado**. `Schedules.specialityId` é `string` simples.

- **“Posso usar Redis no lugar?”**  
  Podes, mas o Redis é volátil. O Shadow em BD dá **persistência**, **auditoria**, **joins** e **reporting**.

- **“E se o micro mudar o nome da especialidade?”**  
  O próximo **refresh**/evento actualiza o Shadow. Os teus relatórios passam a trazer o nome novo.

---

## 11) Checklist de implementação

- [ ] Migration do modelo `SpecialityShadow` criada e aplicada.
- [ ] `ensureSpeciality()` implementado com 422/503 adequados.
- [ ] Sync periódico + logs/metrics.
- [ ] Endpoints de escrita validam `specialityId` via Shadow.
- [ ] Relatórios usam `name` do Shadow quando disponível.
- [ ] Política de TTL/mode degradado definida e documentada.
- [ ] Alertas para falhas de sync/circuit breaker configurados.

---

> **Conclusão:** O padrão *Shadow* no Prisma/BD dá-te **robustez operacional**, **desacoplamento** e **boa UX**,
> sem complicar o domínio. Usa-o de forma disciplinada (dados mínimos, sync previsível, erros claros) e
> ganharás estabilidade mesmo quando os micros vizinhos estiverem em manutenção ou sob carga.
