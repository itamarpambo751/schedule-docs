# Guia de Slots — Implementação (Marcação com horários nos Slots)

Este documento descreve as **regras, invariantes, formato do JSON `hours`** e o **fluxo de geração/merge** de horas para o modelo `Slots`, **alinhado com o schema Prisma mais recente** (inclui campos de visibilidade/publicação e aprovação).

> **UTC sempre:** todas as datas/horas devem ser `timestamptz` em UTC.  
> **Idempotência:** operações de geração/merge NUNCA devem apagar horas ocupadas (BOOKED/HELD) nem quebrar histórico.  
> **Políticas:** `ExcludeDay` tem prioridade sobre `ExcludeRange`.

---

## Modelo Prisma — Slots (atualizado)

```prisma
model Slots {
  id                          String   @id @default(uuid())
  inheritAllProfessionals     Boolean  @default(true)         // herda profissionais do Schedule?
  availableProfessionalTaxIds String[] @default([])           // lista específica quando não herda
  weekDay                     WeekDays                        // dia da semana em UTC
  specificDate                DateTime @db.Timestamptz        // dia do slot (00:00:00Z)
  markingLimit                Int                              // capacidade do dia
  isClosed                    Boolean  @default(false)         // fechado (capacidade atingida)

  genderRestriction GenderRestrictions @default(NONE)

  scheduleId String
  schedule   Schedules @relation(fields: [scheduleId], references: [id], onDelete: Restrict)

  /**
   * Estrutura JSON em `hours` (sempre UTC):
   * [
   *   {
   *     "hour": "2025-10-20T08:00:00Z",
   *     "status": "AVAILABLE" | "HELD" | "BOOKED" | "CANCELLED" | "BLOCKED",
   *     "appointmentId": null,
   *     "professionalTaxId": null,
   *     "roomLink": null,
   *     "remarks": null,
   *     "updatedAt": "2025-10-18T12:00:00Z",
   *     "filledBy": null
   *   }
   * ]
   */
  hours Json

  appointments Appointment[]

  // Publicação & revisão
  visibility       SlotVisibility @default(DRAFT)  // DRAFT | PUBLISHED | REJECTED | ARCHIVED
  approvalRequired Boolean        @default(true)   // exige aprovação?
  submittedByTaxId String?                         // quem submeteu para revisão
  reviewedByTaxId  String?                         // quem aprovou/rejeitou
  reviewedAt       DateTime?      @db.Timestamptz
  publishAt        DateTime?      @db.Timestamptz  // quando foi/será publicado
  reviewNotes      String?

  // Guardas de janela de publicação (opcionais)
  publishNotBefore DateTime? @db.Timestamptz
  publishExpiresAt DateTime? @db.Timestamptz

  createdAt DateTime  @default(now()) @db.Timestamptz
  updatedAt DateTime  @updatedAt @db.Timestamptz
  deletedAt DateTime?

  // Índices & unicidade
  @@unique([scheduleId, specificDate])
  @@index([visibility, specificDate])
  @@index([publishAt])
  @@index([scheduleId, specificDate])
  @@index([scheduleId, weekDay])
  @@index([deletedAt])
}
```

> **Nota:** mantenha o par único `@@unique([scheduleId, specificDate])` para garantir **um slot por dia** para cada agenda.

---

## 1) Invariantes Recomendados

- **Unicidade (um dia por agenda):** `@@unique([scheduleId, specificDate])`  
- **UTC Sempre:** `specificDate` e `hours[i].hour` em UTC  
- **`markingLimit` consistente:**
  - **Via A (grelha por duração):** capacidade = `#timeboxes × (multi-atendimento opcional)`  
  - **Via B (dia-como-slot):** capacidade = `maxAppointmentsPerSlot` do Schedule
- **Fecho automático:** `isClosed = true` quando capacidade atingida / todas as boxes ocupadas
- **Herança de profissionais:** `inheritAllProfessionals = true` → usar `schedule.availableProfessionalTaxIds`

---

## 2) Esquema do JSON `hours`

```json
[
  {
    "hour": "2025-10-20T08:00:00Z",
    "status": "AVAILABLE",
    "isFilled": false,
    "appointmentId": null,
    "professionalTaxId": null,
    "roomLink": null,
    "remarks": null,
    "updatedAt": "2025-10-18T12:00:00Z",
    "filledBy": null
  }
]
```

### Derivação sugerida de `isFilled` a partir de `status`

| status     | isFilled |
|-----------:|:--------:|
| AVAILABLE  | false    |
| HELD       | true     |
| BOOKED     | true     |
| CANCELLED  | false    |
| BLOCKED    | true     |

---

## 3) Geração & MERGE de horas

### 3.1 Princípios
- **NUNCA** sobrescrever/remover horas **BOOKED/HELD/BLOCKED**  
- **Adicionar** novas horas quando a grelha aumentar  
- **Remover** apenas horas **AVAILABLE/CANCELLED** que ficaram fora do novo cálculo  
- **UTC** em toda a pipeline

### 3.2 Tipos/Helpers (TypeScript)

```ts
type HourBox = {
  hour: string; // ISO UTC
  status: 'AVAILABLE'|'HELD'|'BOOKED'|'CANCELLED'|'BLOCKED';
  appointmentId?: string|null;
  professionalTaxId?: string|null;
  roomLink?: string|null;
  remarks?: string|null;
  updatedAt?: string;
  filledBy?: string|null;
};

/** Merge idempotente & seguro */
function mergeHoursSafely(existing: HourBox[], incoming: HourBox[]): HourBox[] {
  const byKey = new Map(existing.map(h => [h.hour, h]));
  for (const h of incoming) {
    const cur = byKey.get(h.hour);
    if (!cur) { byKey.set(h.hour, h); continue; }
    const occupied = ['BOOKED','HELD','BLOCKED'].includes(cur.status);
    if (occupied) continue;
    byKey.set(h.hour, { ...cur, ...h, updatedAt: new Date().toISOString() });
  }
  const incomingSet = new Set(incoming.map(h => h.hour));
  return Array.from(byKey.values()).filter(h => {
    const keepOccupied = ['BOOKED','HELD','BLOCKED'].includes(h.status);
    if (keepOccupied) return true;
    return incomingSet.has(h.hour);
  });
}

/** Projeta HH:mm:ss do Schedule numa data específica (UTC) */
function projectScheduleTimeToDate(scheduleTimeUtc: Date, dayUtc: Date) {
  return new Date(Date.UTC(
    dayUtc.getUTCFullYear(),
    dayUtc.getUTCMonth(),
    dayUtc.getUTCDate(),
    scheduleTimeUtc.getUTCHours(),
    scheduleTimeUtc.getUTCMinutes(),
    scheduleTimeUtc.getUTCSeconds()
  ));
}

/** Grelha via duração (Via A) */
function buildHoursGrid(schedule: {
  startTime: string | Date;
  endTime: string | Date;
  appointmentDurationInMinutes: number;
}, dayUtc: Date): HourBox[] {
  if (!schedule.appointmentDurationInMinutes || schedule.appointmentDurationInMinutes <= 0) {
    throw new Error('buildHoursGrid requer appointmentDurationInMinutes > 0');
  }
  const start = projectScheduleTimeToDate(new Date(schedule.startTime), dayUtc);
  const end   = projectScheduleTimeToDate(new Date(schedule.endTime), dayUtc);
  if (+end <= +start) throw new Error('startTime deve ser anterior a endTime');

  const stepMs = Math.floor(schedule.appointmentDurationInMinutes) * 60_000;
  const out: HourBox[] = [];
  for (let t = +start; t < +end; t += stepMs) {
    out.push({ hour: new Date(t).toISOString(), status: 'AVAILABLE', appointmentId: null, professionalTaxId: null, updatedAt: new Date().toISOString() });
  }
  return out;
}

/** Grelha via capacidade (Via B — dia como slot) */
function buildCapacityBoxes(schedule: { maxAppointmentsPerSlot?: number|null }, dayUtc: Date): HourBox[] {
  const cap = schedule.maxAppointmentsPerSlot && schedule.maxAppointmentsPerSlot > 0 ? schedule.maxAppointmentsPerSlot : 1;
  const base = new Date(Date.UTC(dayUtc.getUTCFullYear(), dayUtc.getUTCMonth(), dayUtc.getUTCDate(), 0, 0, 0));
  return Array.from({ length: cap }, (_, i) => ({
    hour: new Date(+base + i).toISOString(),
    status: 'AVAILABLE',
    appointmentId: null,
    professionalTaxId: null,
    updatedAt: new Date().toISOString()
  }));
}

/** Exclusões (stub) — aplicar ExcludeDay > ExcludeRange */
function applyExcludeRanges(baseHours: HourBox[], excludeRanges: any[], dayUtc: Date): HourBox[] {
  // Implementar conforme políticas locais:
  // - Remover/recortar janelas (12:00–13:00) e datas pontuais
  // - Honrar RRULE
  // - Nunca remover BOOKED/HELD; marcar como BLOCKED se necessário
  return baseHours;
}

/** Capacidade & fecho */
function deriveMarkingLimit(schedule: any, hours: HourBox[]): number {
  if (typeof schedule.maxAppointmentsPerSlot === 'number' && schedule.maxAppointmentsPerSlot > 0) return schedule.maxAppointmentsPerSlot;
  return hours.length;
}
function inferCapacityFromGrid(hours: HourBox[]): number { return hours.length; }
function countFilled(hours: HourBox[]): number { return hours.filter(h => ['BOOKED','HELD'].includes(h.status)).length; }
function allTimeBoxesFilled(hours: HourBox[]): boolean { return hours.every(h => ['BOOKED','HELD','BLOCKED'].includes(h.status)); }
function weekday(d: Date): 'SUNDAY'|'MONDAY'|'TUESDAY'|'WEDNESDAY'|'THURSDAY'|'FRIDAY'|'SATURDAY' {
  return ['SUNDAY','MONDAY','TUESDAY','WEDNESDAY','THURSDAY','FRIDAY','SATURDAY'][d.getUTCDay()] as any;
}
```

---

## 4) Upsert idempotente do Slot (por dia)

```ts
async function upsertSlotDay({
  prisma,
  schedule,
  dateUtc,
  inheritAll
}: {
  prisma: any; // PrismaClient
  schedule: {
    id: string;
    startTime: string | Date;
    endTime: string | Date;
    appointmentDurationInMinutes?: number | null;
    maxAppointmentsPerSlot?: number | null;
    availableProfessionalTaxIds?: string[] | null;
    excludeRanges?: any[];
  };
  dateUtc: Date;
  inheritAll: boolean;
}) {
  const usesDuration = !!schedule.appointmentDurationInMinutes && schedule.appointmentDurationInMinutes > 0;
  const baseHours = usesDuration
    ? buildHoursGrid(schedule as any, dateUtc)
    : buildCapacityBoxes(schedule as any, dateUtc);

  const hoursAfterExclude = applyExcludeRanges(baseHours, schedule.excludeRanges ?? [], dateUtc);

  const existing = await prisma.slots.findUnique({
    where: { scheduleId_specificDate: { scheduleId: schedule.id, specificDate: dateUtc } }
  });

  const base = existing ?? {
    scheduleId: schedule.id,
    specificDate: dateUtc,
    weekDay: weekday(dateUtc),
    inheritAllProfessionals: inheritAll,
    availableProfessionalTaxIds: inheritAll ? (schedule.availableProfessionalTaxIds ?? []) : [],
    markingLimit: deriveMarkingLimit(schedule as any, hoursAfterExclude),
    isClosed: false,
    hours: [] as HourBox[]
  };

  const merged = mergeHoursSafely(base.hours as HourBox[], hoursAfterExclude);
  const capacity = base.markingLimit ?? inferCapacityFromGrid(merged);
  const isClosed = countFilled(merged) >= capacity || allTimeBoxesFilled(merged);

  if (existing) {
    await prisma.slots.update({
      where: { id: existing.id },
      data: {
        hours: merged,
        isClosed,
        availableProfessionalTaxIds: base.inheritAllProfessionals
          ? (schedule.availableProfessionalTaxIds ?? [])
          : base.availableProfessionalTaxIds
      }
    });
    return existing.id;
  }

  const created = await prisma.slots.create({ data: { ...base, hours: merged, isClosed } });
  return created.id;
}
```

---

## 5) Exclusões

**Prioridade:** `ExcludeDay > ExcludeRange`  
- `ExcludeDay` → remove o dia inteiro  
- `ExcludeRange` → corta/remarca intervalos

**Regra de ouro:**  
- `BOOKED/HELD` → **nunca remover**  
- `AVAILABLE/CANCELLED` → podem ser removidos ou marcados como `BLOCKED`

---

## 6) Validações rápidas

- `specificDate` em UTC  
- `weekDay === weekday(specificDate)`  
- `inheritAllProfessionals = true` → ignorar `availableProfessionalTaxIds` do payload  
- `hours[*].hour` sem duplicados e **sempre ISO-8601 UTC**  
- Em reserva, validar se `professionalTaxId` pertence à lista herdada ou específica do slot

---

## 7) Índices e Constraints

```prisma
@@unique([scheduleId, specificDate])
@@index([visibility, specificDate])
@@index([publishAt])
@@index([scheduleId, specificDate])
@@index([scheduleId, weekDay])
@@index([deletedAt])
```

---

## 8) Fluxo de Reserva

1. Selecionar `hour` no JSON do Slot  
2. Validar `status === 'AVAILABLE'` e autorização do profissional  
3. Criar `Appointment` e atualizar `hours[i]` (`status → BOOKED`, `appointmentId`)  
4. Atualizar `isClosed` conforme capacidade  
5. Em conflito → `409 Conflict`

---

## 9) Erros Comuns e Códigos

| Situação                                       | Código | Mensagem                                             |
|-----------------------------------------------|--------|------------------------------------------------------|
| Duplicar slot no mesmo dia/agenda             | 409    | Slot for this schedule and date already exists       |
| `hours[*].hour` inválido ou não UTC           | 422    | hours[*].hour must be ISO-8601 UTC                  |
| Misturar Via A e Via B                        | 422    | Provide appointmentDurationInMinutes or maxAppointmentsPerSlot |
| Remover hora com marcação/hold                | 409    | Cannot remove booked/held hour                      |
| Profissional não autorizado                   | 403    | Professional not allowed for this slot              |
| `weekDay` não corresponde a `specificDate`    | 422    | weekDay must match specificDate                     |
| Falha de idempotência/concurrency             | 409    | Slot generation already in progress                 |

---

## 10) Checklist de Testes

- Geração **Via A** (duração) e **Via B** (capacidade)  
- Merge preserva **BOOKED/HELD/BLOCKED**  
- Exclusões respeitam **ExcludeDay > ExcludeRange**  
- Projeção UTC correta em virada de mês/ano  
- Herança de profissionais (on/off)  
- `isClosed` liga/desliga conforme capacidade  
- Reserva concorrente → `409 Conflict`  
- Soft-delete não afeta consultas históricas
