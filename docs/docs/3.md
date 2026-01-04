
# 3. Exclusões e Folgas

No sistema, **nem todos os dias e horários de uma agenda estarão disponíveis para marcação**. Isso pode ocorrer por:

- Feriados
- Reuniões ou eventos internos da unidade
- Férias ou folgas de profissionais
- Bloqueios específicos de especialidade

Esses bloqueios são tratados por **ExcludeDay** (dia específico) e **ExcludeRange** (período de tempo) e se aplicam automaticamente aos **slots** gerados.

---

## 3.1 ExcludeDay (Bloqueio de Dia Específico)

O **ExcludeDay** serve para **bloquear datas inteiras**, independentemente dos horários internos.

### Componentes:

- **SpecificDate**: data exata do bloqueio (ex: 07/09/2024)
- **WeekDays**: se for recorrente semanalmente (ex: toda segunda-feira)
- **TypeOfRecurrence**: diária, semanal, mensal, personalizada
- **ExclusionVisibility**: quem vê ou é afetado (sistema, unidade, especialidade)
- **Reason**: motivo do bloqueio
- **DefinedBy**: quem criou o bloqueio

### Exemplo prático:

- **Motivo:** Feriado nacional
- **Data:** 07/09/2024
- **Visibilidade:** Todos (SYSTEM)
- **Ação do sistema:**

  - Todos os slots criados para 07/09/2024 ficam bloqueados
  - HourBoxes desse dia não podem ser reservados
  - Se já houver consultas, sistema notifica pacientes e profissionais

---

## 3.2 ExcludeRange (Bloqueio de Período ou Horário)

O **ExcludeRange** permite **bloquear horários específicos**, dentro de um ou vários dias.

### Componentes:

- **StartTime / EndTime**: intervalo de horas (ex: 14:00-16:00)
- **StartDate / EndDate**: período do bloqueio (ex: 15/07 a 30/07)
- **ExcludeForAllSlots**: se deve afetar todos os slots do dia
- **ExcludeFor**: dias da semana específicos
- **ExcludeForSpecificDates**: datas exatas
- **TypeOfRecurrence**: diária, semanal, mensal, personalizada
- **ExclusionVisibility** e **Reason**

### Exemplo prático:

- **Motivo:** Reunião da equipe
- **Período:** 14:00 às 15:00
- **Recorrência:** semanal, toda segunda-feira
- **Impacto:**

  - Slots gerados para segunda-feira terão HourBoxes entre 14:00 e 15:00 **bloqueados automaticamente**
  - Pacientes não podem reservar esses horários
  - Profissionais são notificados do bloqueio

---

## 3.3 Folgas e Férias de Profissionais

O sistema também integra **trabalho dos profissionais** com exclusões:

1. Cada profissional possui **horários de trabalho (WorkingHour)**.
2. Se o profissional estiver de **folga ou férias**, o sistema cria automaticamente **ExcludeRange** ou **ExcludeDay** para os slots que coincidem com os horários do profissional.
3. Ao gerar os slots da agenda, o sistema **atribui apenas os profissionais disponíveis** em cada HourBox.

### Exemplo prático:

- Dra. Ana (Cardiologista) trabalha:
  - Segunda a sexta: 13:00-18:00
  - Sábado: 08:00-12:00
- Férias: 15/07/2024 a 30/07/2024

Resultado do sistema:

- Exclusões automáticas criadas para esse período
- Slots gerados:
  - Sem Dra. Ana
  - Se houver outros profissionais, eles serão atribuídos
  - Se Dra. Ana for a única profissional, o slot pode ser **bloqueado**

---

## 3.4 Interação com Slots e HourBoxes

- **Slots já gerados** podem ser **atualizados automaticamente** ao criar uma exclusão.
- HourBoxes afetados pelo bloqueio:
  - Se estiverem **AVAILABLE**, mudam para **BLOCKED**
  - Se já estiverem **BOOKED**, o sistema notifica o paciente e sugere remarcação
- **Tipos de visibilidade** permitem aplicar o bloqueio apenas a:
  - Sistema inteiro
  - Unidade de saúde
  - Especialidade específica
  - Área médica

---

## 3.5 Exemplos combinados

### Cenário 1: Feriado + Profissional de Folga

- Agenda de Pediatria: Segunda a Sexta, 08:00-12:00
- Feriado: 15/04/2024
- Dra. Ana de férias: 14/04 a 18/04

**Resultado:**

| Data       | Slot Status | HourBoxes disponíveis | Profissional atribuído |
| ---------- | ----------- | --------------------- | ---------------------- |
| 14/04/2024 | Aberto      | 08:00-12:00           | Dr. Carlos             |
| 15/04/2024 | Bloqueado   | Todos                 | Nenhum                 |
| 16/04/2024 | Aberto      | 08:00-12:00           | Dr. Carlos             |
| 17/04/2024 | Aberto      | 08:00-12:00           | Dr. Carlos             |
| 18/04/2024 | Aberto      | 08:00-12:00           | Dr. Carlos             |

### Cenário 2: Bloqueio parcial (Reunião da equipe)

- Bloqueio: Segunda-feira, 14:00-15:00
- Slot: 15/04/2024, 08:00-16:00, capacidade 2 por horário

**Resultado HourBoxes:**

| Hora  | Status    |
| ----- | --------- |
| 08:00 | AVAILABLE |
| 08:30 | AVAILABLE |
| ...   | ...       |
| 14:00 | BLOCKED   |
| 14:30 | BLOCKED   |
| 15:00 | AVAILABLE |
| ...   | ...       |

> O sistema garante que apenas horários válidos fiquem disponíveis, mesmo que a agenda seja recorrente.

