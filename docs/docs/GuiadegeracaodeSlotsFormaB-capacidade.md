# Regras claras de “modo capacidade” (sem horários)

---

## Detecção do modo

| Modo | Condição | hours |
|------|-----------|--------|
| **TIMED (com horários)** | `appointmentDurationInMinutes > 0` | preenchido |
| **CAPACITY (sem horários)** | `appointmentDurationInMinutes == null` **e** `maxAppointmentsPerSlot > 0` | `[]` (vazio) |

---

## Geração de Slots

### Em CAPACITY:
- `slots.hours = []` (array vazio)
- `slots.markingLimit = schedules.maxAppointmentsPerSlot`
- `slots.isClosed = (totalAppointmentsDoDia >= markingLimit)`
- `inheritAllProfessionals` continua a funcionar (validação de quem pode atender)

---

## Reservas (Appointment)

### Em CAPACITY:
- `Appointment.selectedHour = null`
- Obrigatórios:
  - `slotId`
  - `patientTaxId`
  - `professionalTaxId`
  - `typeOfService`

A lógica de capacidade é por **contagem de appointments do slot**.

> `@@unique([slotId, selectedHour, professionalTaxId])` não bloqueia (NULL em Postgres não conflita).

---

## Invariantes

Se a agenda estiver em **CAPACITY**:

- `slots.hours` deve estar vazio (não gerar timeboxes)
- `markingLimit > 0`
- Ao criar appointment → **não ultrapassar o markingLimit**

---

## Ajustes no serviço (geração e reserva)

### A) Geração de Slot (CAPACITY)

```ts
/**
 * Função responsável por gerar (ou atualizar) um Slot diário no modo CAPACITY.
 * Esse modo não possui horários (hours = []), apenas um limite de marcações por dia.
 */
async function upsertSlotDay_Capacity({
  prisma,
  schedule,
  dateUtc,
  inheritAll
}: {
  prisma: any; // Instância do Prisma para acesso ao banco de dados
  schedule: {
    id: string;                         // ID da agenda (schedule)
    maxAppointmentsPerSlot: number;     // Limite máximo de marcações por slot (capacidade)
    availableProfessionalTaxIds?: string[]; // Lista de profissionais permitidos (opcional)
  };
  dateUtc: Date;       // Data do dia que se deseja criar/atualizar o slot
  inheritAll: boolean; // Se verdadeiro, o slot herda todos os profissionais da agenda
}) {
  // 1️⃣ No modo CAPACITY, não há horários — apenas um array vazio.
  const hours: any[] = [];

  // 2️⃣ O limite de marcações (capacidade) vem diretamente da agenda.
  const markingLimit = schedule.maxAppointmentsPerSlot;

  // 3️⃣ Verifica se já existe um Slot criado para essa agenda e data específica.
  //    A busca usa um índice composto (scheduleId + specificDate).
  const existing = await prisma.slots.findUnique({
    where: { scheduleId_specificDate: { scheduleId: schedule.id, specificDate: dateUtc } }
  });

  // 4️⃣ Define os dados base para criação (ou atualização) do Slot.
  //    - `inheritAllProfessionals`: indica se o slot herda todos os profissionais da agenda.
  //    - `availableProfessionalTaxIds`: lista de profissionais válidos (ou vazia).
  //    - `markingLimit`: capacidade máxima de marcações por dia.
  //    - `isClosed`: indica se o slot está cheio (por padrão começa como false).
  //    - `hours`: vazio (pois modo CAPACITY não possui timeboxes).
  const baseData = {
    scheduleId: schedule.id,
    specificDate: dateUtc,
    weekDay: weekday(dateUtc), // Converte a data para dia da semana (MONDAY, TUESDAY, etc.)
    inheritAllProfessionals: inheritAll,
    availableProfessionalTaxIds: inheritAll ? (schedule.availableProfessionalTaxIds ?? []) : [],
    markingLimit,
    isClosed: false,
    hours
  };

  // 5️⃣ Caso já exista um Slot para o mesmo dia...
  if (existing) {
    // 5.1️⃣ Conta quantos appointments (marcações) existem nesse slot.
    //      Marcações deletadas (deletedAt != null) não contam.
    const count = await prisma.appointment.count({ where: { slotId: existing.id, deletedAt: null } });

    // 5.2️⃣ Verifica se a capacidade foi atingida.
    const isClosed = count >= (existing.markingLimit ?? markingLimit);

    // 5.3️⃣ Atualiza o Slot existente com os novos dados.
    //      - Garante que `hours` fique vazio.
    //      - Atualiza o `markingLimit` e `isClosed`.
    //      - Atualiza a lista de profissionais disponíveis (se herdar da agenda).
    await prisma.slots.update({
      where: { id: existing.id },
      data: {
        hours, // limpa possíveis horários antigos (caso tenha mudado de modo TIMED → CAPACITY)
        markingLimit,
        isClosed,
        availableProfessionalTaxIds: baseData.inheritAllProfessionals
          ? (schedule.availableProfessionalTaxIds ?? [])
          : existing.availableProfessionalTaxIds
      }
    });

    // 5.4️⃣ Retorna o ID do Slot atualizado.
    return existing.id;
  }

  // 6️⃣ Caso o Slot ainda não exista:
  //     Cria um novo registro com base nos dados preparados.
  const created = await prisma.slots.create({ data: baseData });

  // 7️⃣ Retorna o ID do Slot recém-criado.
  return created.id;
}

/**
 * Função auxiliar que converte uma data UTC para o nome do dia da semana.
 * Exemplo: segunda → "MONDAY"
 */
function weekday(d: Date) {
  return ['SUNDAY','MONDAY','TUESDAY','WEDNESDAY','THURSDAY','FRIDAY','SATURDAY'][d.getUTCDay()];
}
```

# Função: upsertSlotDay_Capacity

## Explicação Geral

### Objetivo
Criar ou atualizar o registro de um **Slot diário** (um dia específico de atendimento) para uma agenda que opera em **modo capacidade**.

### Modo CAPACITY
Diferente do modo tradicional (com horários), ele **não cria subdivisões de tempo** — apenas define **quantas marcações cabem por dia**.

### Fluxo
1. Verifica se já existe um slot para a data → **atualiza** se sim.  
2. Se não existe → **cria** um novo slot.  
3. Garante que o campo **`hours`** fique sempre **vazio**.  
4. Controla o campo **`isClosed`** com base no número de marcações (`appointment.count`).


---

### B) Criar Appointment em CAPACITY (sem selectedHour)

Objetivo: **respeitar o `markingLimit` do dia sem timeboxes.**

# Função: createAppointmentCapacityMode

## Explicação Geral

### Objetivo
Criar uma **nova marcação (appointment)** em um **slot do tipo capacidade**, garantindo que:
- O limite máximo de marcações do dia não seja ultrapassado.  
- Apenas profissionais autorizados possam registrar consultas.  
- O status do slot seja atualizado (fechado/aberto) conforme o número de marcações.

### Modo CAPACITY
Este modo **não utiliza subdivisões de horários** (`hours` fica vazio).  
Cada slot representa **um único dia** e possui um **limite de atendimentos**.

### Fluxo
1. **Bloqueia o slot** (com update sem alteração real) para evitar race conditions.  
2. **Valida** se o slot está configurado corretamente para modo capacidade.  
3. **Confere** se o profissional é permitido (caso `inheritAllProfessionals` seja true).  
4. **Conta** quantas marcações já existem no slot.  
5. **Impede** criação se a capacidade já foi atingida.  
6. **Cria** a nova marcação (`appointment`).  
7. **Atualiza** o status do slot (fecha se o limite for atingido).  

---

## Código detalhado

```ts
// Função assíncrona responsável por criar uma marcação (appointment)
// em um slot configurado no modo CAPACITY (sem subdivisões de horário).
async function createAppointmentCapacityMode(prisma: any, {
  slotId,              // ID do slot do dia onde será feita a marcação
  patientTaxId,        // Identificador fiscal do paciente
  professionalTaxId,   // Identificador fiscal do profissional que atenderá
  typeOfService,       // Tipo de atendimento: presencial, telemedicina ou domiciliar
  scheduleId,          // ID da agenda (opcional)
  notes                // Notas adicionais (opcional)
}: {
  slotId: string;
  patientTaxId: string;
  professionalTaxId: string;
  typeOfService: 'IN_PERSON'|'TELEMEDICINE'|'HOME';
  scheduleId?: string|null;
  notes?: string|null;
}) {

  // Inicia uma transação para garantir integridade dos dados
  // — todas as operações abaixo ocorrem de forma atômica.
  return await prisma.$transaction(async (tx: any) => {

    // Atualiza o campo "updatedAt" do slot apenas para forçar um "lock" de linha.
    // Isso previne condições de corrida quando múltiplas marcações ocorrem simultaneamente.
    const slot = await tx.slots.update({
      where: { id: slotId },
      data: { updatedAt: new Date() } // operação "no-op"
    });

    // Valida se o slot possui subdivisões de horário.
    // Caso tenha, este slot não deveria usar o modo CAPACITY.
    if (Array.isArray(slot.hours) && slot.hours.length > 0) {
      throw makeHttpError(409, 'Slot requires timed booking (hours present)');
    }

    // Caso o slot herde todos os profissionais, verifica se o atual é permitido.
    if (slot.inheritAllProfessionals) {
      const allowed = (slot.availableProfessionalTaxIds ?? []).includes(professionalTaxId);
      if (!allowed)
        throw makeHttpError(403, 'Professional not allowed for this slot');
    }

    // Conta quantas marcações já existem neste slot (que não foram deletadas).
    const currentCount = await tx.appointment.count({
      where: { slotId: slot.id, deletedAt: null }
    });

    // Caso o número de marcações atinja o limite máximo do slot, marca-o como fechado.
    if (currentCount >= slot.markingLimit) {
      await tx.slots.update({
        where: { id: slot.id },
        data: { isClosed: true }
      });
      throw makeHttpError(409, 'Slot capacity reached');
    }

    // Cria uma nova marcação (appointment) associada ao slot.
    const appt = await tx.appointment.create({
      data: {
        slotId: slot.id,                // Vincula ao slot do dia
        selectedHour: null,             // Null no modo CAPACITY (sem horário específico)
        patientTaxId,                   // Identificador do paciente
        professionalTaxId,              // Identificador do profissional
        typeOfService,                  // Tipo de atendimento
        status: 'PENDING',              // Status inicial
        scheduleId: scheduleId ?? null, // Agenda associada, se houver
        notes: notes ?? null            // Observações adicionais
      }
    });

    // Após criar, incrementa o contador e verifica se o slot deve ser fechado.
    const afterCount = currentCount + 1;
    const willClose = afterCount >= slot.markingLimit;

    // Atualiza o estado do slot (fecha se atingiu a capacidade).
    if (willClose || slot.isClosed !== willClose) {
      await tx.slots.update({
        where: { id: slot.id },
        data: { isClosed: willClose }
      });
    }

    // Retorna a marcação criada.
    return appt;
  });
}

// Função auxiliar que cria um erro HTTP com status e mensagem personalizados.
// Usada para padronizar os erros retornados em caso de falhas de regra de negócio.
function makeHttpError(status: number, message: string) {
  const err: any = new Error(message);
  err.status = status;
  return err;
}
```

---

## Notas importantes

- O **“lock” via update** garante serialização suficiente para o contador (evita overbooking).
- Em Postgres, o ideal seria `SELECT ... FOR UPDATE`, mas com Prisma o padrão é usar **update “no-op”** dentro da transação.

---

## Validações extra recomendadas

### Na criação/atualização da Agenda (Schedules):
- Se `appointmentDurationInMinutes == null` → exigir `maxAppointmentsPerSlot > 0`
- Se `appointmentDurationInMinutes > 0` → `maxAppointmentsPerSlot` pode ser vazio

### Na geração de Slots:
- Em CAPACITY → `hours` vazio e `markingLimit > 0`

### Na reserva:
- Em CAPACITY → rejeitar `selectedHour` (422 “selectedHour must be null in capacity mode”)
- Validar `professionalTaxId` conforme `inheritAllProfessionals` e `availableProfessionalTaxIds`

---

## Erros e códigos (modo capacidade)

| Onde | Regra | Código | Mensagem |
|------|--------|--------|-----------|
| Criar Agenda | `appointmentDurationInMinutes == null` e `maxAppointmentsPerSlot <= 0` | 422 | Provide maxAppointmentsPerSlot when appointmentDurationInMinutes is null |
| Gerar Slot | CAPACITY mas `hours` veio preenchido | 422 | hours must be empty for capacity mode |
| Reservar | Slot é TIMED mas endpoint CAPACITY foi usado | 409 | Slot requires timed booking (hours present) |
| Reservar | Profissional não autorizado | 403 | Professional not allowed for this slot |
| Reservar | Capacidade atingida | 409 | Slot capacity reached |
| Reservar | `selectedHour` enviado em CAPACITY | 422 | selectedHour must be null for capacity mode |

---

## Sobre índices/constraints

- `@@unique([slotId, selectedHour, professionalTaxId])` **não bloqueia CAPACITY**
  - `selectedHour` é `NULL` → Postgres permite múltiplos `NULLs`
- A limitação de capacidade é controlada **na lógica transacional**, não no índice

---

## Recap

No modo **CAPACITY**:
- `Slot` não tem horários → `hours = []`
- `Appointment` usa apenas `slotId`, com `selectedHour = null`
- A capacidade é controlada por `Slots.markingLimit`
- O `markingLimit` vem de `Schedules.maxAppointmentsPerSlot`
- A verificação é transacional para evitar overbooking
