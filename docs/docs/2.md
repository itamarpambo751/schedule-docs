
# 2. Conceitos-base do Sistema

## 2.1 Agenda

A **Agenda** é a **regra-mestra do agendamento**. Ela define **quando e como o atendimento ocorre**.

### Componentes de uma Agenda:

- **Unidade de Saúde**: hospital, clínica ou consultório onde ocorre o atendimento.
- **Especialidade / Categoria**: Pediatria, Clínica Geral, Dermatologia, Exames, Vacinas.
- **Dias da Semana**: Segunda a sexta, apenas terças, etc.
- **Horário de Atendimento**: início e fim do expediente.
- **Duração da Consulta**: ex: 30 minutos por paciente.
- **Capacidade por horário**: quantos pacientes podem ser atendidos simultaneamente.
- **Recorrência**: define se a agenda se repete em dias futuros (diária, semanal, mensal, etc.).
- **Tipo de Serviço**: presencial, telemedicina ou domiciliar.
- **Restrições**: idade, gênero, especialidade do paciente.
- **Profissionais disponíveis**: lista de médicos que podem atender dentro da agenda.
- **Exclusões**: dias ou horários que devem ser bloqueados (feriados, reuniões, férias de profissionais).

### Exemplo prático de Agenda

Suponha que o **Hospital São Lucas** quer criar uma agenda de **Pediatria** para o **Dr. Carlos**:

- **Dias da semana**: Segunda, Quarta e Sexta
- **Horário**: 08:00 às 12:00
- **Duração da consulta**: 30 minutos
- **Capacidade por horário**: 2 pacientes
- **Tipo de serviço**: Presencial
- **Restrições de idade**: até 18 anos
- **Profissionais disponíveis**: Dr. Carlos, Dra. Ana
- **Recorrência**: semanal (agenda se repete toda segunda, quarta e sexta até uma data final)

Com essas regras, a agenda **não cria horários ainda**, ela apenas define **a forma e regras que os slots e horários seguirão**.

---

## 2.2 Slot

O **Slot** representa **um dia específico dentro da agenda**.

> Diferente do HourBox, o slot **não é um horário**, mas sim **o “container” que conterá todos os horários daquele dia**, respeitando a duração da consulta, capacidade e profissionais disponíveis.

### Componentes do Slot:

- **Data específica** (ex: 15/04/2024) ou calculada a partir de uma agenda recorrente.
- **Capacidade do dia**: quantos pacientes podem ser atendidos por horário.
- **Profissionais atribuídos**: automaticamente com base na disponibilidade definida no horário de trabalho de cada profissional.
- **Status do Slot**: aberto, fechado ou bloqueado.
- **Visibilidade**: se o slot já pode aparecer para marcação ou ainda está em rascunho.
- **Exclusões**: feriados, reuniões ou folgas que afetam aquele dia específico.

### Exemplo prático de Slot

- Agenda: Pediatria, Segunda, 08:00-12:00, 30 minutos de duração
- Slot gerado: Segunda, 15/04/2024
- Horários dentro do Slot (HourBoxes): 08:00, 08:30, 09:00, 09:30, 10:00, 10:30, 11:00, 11:30
- Cada HourBox terá **capacidade para 2 pacientes**
- Profissionais atribuídos: Dr. Carlos (disponível nesse dia e horário)

> Observação: se houver uma exclusão, como feriado em 15/04/2024, o slot será **bloqueado automaticamente**, e os horários não estarão disponíveis para marcação.

---

## 2.3 HourBox (Horário Real de Consulta)

O **HourBox** é o **horário específico dentro do slot**, que pode ser reservado por um paciente.

### Componentes do HourBox:

- **Hora específica** (ex: 08:00)
- **Status**:
  - AVAILABLE: disponível para marcação
  - BOOKED: reservado com confirmação
  - HELD: temporariamente reservado (aguardando pagamento ou aprovação)
  - CANCELLED: cancelado
  - BLOCKED: indisponível por exclusão ou regra da agenda
- **Capacidade**: quantos pacientes podem ser atendidos simultaneamente
- **Profissional atribuído**: automaticamente baseado na agenda e no horário de trabalho
- **Link da telemedicina** (quando aplicável)

### Exemplo de HourBox

Slot: 15/04/2024, 08:00 às 12:00, duração 30 minutos, capacidade 2

| Hora  | Status    | Profissional | Capacidade |
| ----- | --------- | ------------ | ---------- |
| 08:00 | AVAILABLE | Dr. Carlos   | 2          |
| 08:30 | BOOKED    | Dr. Carlos   | 2          |
| 09:00 | AVAILABLE | Dra. Ana     | 2          |

> Cada paciente reserva um HourBox, garantindo que não haja conflito.

---

## 2.4 Recorrência

A **recorrência** define **como a agenda se repete no tempo**, criando automaticamente slots futuros.

### Tipos de recorrência:

- **Diária**: agenda se repete todos os dias úteis
- **Semanal**: agenda se repete em determinados dias da semana (segunda, quarta e sexta)
- **Mensal**: agenda se repete em um dia específico do mês
- **Anual ou personalizada**: para datas especiais ou períodos sazonais

### Impacto da Recorrência

1. Gera automaticamente **slots futuros** conforme a regra.
2. Reduz trabalho manual: não é necessário criar cada dia individualmente.
3. Permite **aplicar exclusões**: feriados, reuniões ou folgas podem sobrepor slots gerados.

### Exemplo prático:

- Agenda semanal: Segunda e Quinta, 08:00-12:00, duração 30 minutos
- Recorrência semanal
- Sistema gera slots automaticamente:
  - 08/04/2024 (Segunda)
  - 11/04/2024 (Quinta)
  - 15/04/2024 (Segunda)
  - 18/04/2024 (Quinta)
- Cada slot terá seus próprios HourBoxes

---

## 2.5 Tipos de Slots (Capacidade x Timed)

O **modo do slot** define **como os horários internos são tratados**:

1. **Timed (Horários fixos)**:
   - Cada HourBox tem horário específico, ex: 08:00, 08:30, 09:00
   - Limita capacidade por horário
   - Ideal para consultas padrão, telemedicina ou exames

2. **Capacity (Baseado em capacidade total do dia)**:
   - Horários não importam, apenas limite de pacientes por slot
   - Sistema distribui consultas internamente
   - Ideal para vacinas ou exames rápidos

3. **Unlimited**:
   - Sem limite de marcações
   - Usado quando não há restrição de número de pacientes


