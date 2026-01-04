

# 6. Exclusões de Dias e Períodos (ExcludeDay / ExcludeRange)

No sistema, nem todos os dias ou horários de um slot podem estar disponíveis para marcação. Para gerenciar isso, usamos **ExcludeDay** (dias específicos ou recorrentes) e **ExcludeRange** (períodos com horário definido).

Essas entidades garantem que **feriados, reuniões, férias ou eventos especiais** não sejam acidentalmente agendados.

---

## 6.1 ExcludeDay (Bloqueio de Dia Específico ou Recorrente)

### Estrutura principal

| Campo               | Descrição                                                                |
| ------------------- | ------------------------------------------------------------------------ |
| SpecificDate        | Data específica a ser bloqueada (ex: 07/09/2024)                         |
| WeekDays            | Dias da semana a bloquear (ex: toda segunda-feira)                       |
| TypeOfRecurrence    | Recorrência: NONE, DAILY, WEEKLY, MONTHLY, ANNUALLY, CUSTOM              |
| ExclusionVisibility | Quem vê/aplica a exclusão: SYSTEM, HEALTH_UNIT, MEDICAL_AREA, SPECIALITY |
| DefinedBy           | Usuário que definiu a exclusão                                           |
| Title / Reason      | Nome e motivo da exclusão (ex: Feriado, Reunião)                         |
| IsActive            | Status da exclusão (ativa ou não)                                        |

---

### 6.1.1 Exemplo Prático: Feriado Nacional

* Bloquear 07/09/2024 (Independência do Brasil)
* Visibilidade: **HEALTH_UNIT**
* Usuário: administrador da unidade

**Resultado:**

* Todos os slots deste dia ficam indisponíveis
* Pacientes não conseguem marcar
* Profissionais recebem notificação de bloqueio

---

### 6.1.2 Exemplo Prático: Reunião Semanal da Equipe

* Bloqueio: **Toda segunda-feira**
* Horário: 14:00 – 15:00 (usa ExcludeRange também)
* Recorrência: **WEEKLY**
* Motivo: "Reunião da equipe"

**Resultado:**

* Sistema automaticamente bloqueia os horários definidos
* Não é necessário criar manualmente todos os dias

---

## 6.2 ExcludeRange (Bloqueio de Período com Hora Específica)

Enquanto o **ExcludeDay** bloqueia dias inteiros, o **ExcludeRange** permite **bloquear horários específicos dentro de um dia ou período**.

### Estrutura principal

| Campo                                | Descrição                                                |
| ------------------------------------ | -------------------------------------------------------- |
| StartTime / EndTime                  | Hora de início e fim do bloqueio                         |
| StartDate / EndDate                  | Período de datas que se aplica                           |
| ExcludeForAllSlots                   | Se bloqueia todos os slots do dia ou apenas selecionados |
| ExcludeFor / ExcludeForSpecificDates | Dias da semana ou datas específicas                      |
| TypeOfRecurrence                     | Recorrência do bloqueio                                  |
| ExclusionVisibility                  | Sistema, unidade, especialidade, etc.                    |
| DefinedBy                            | Usuário que criou a exclusão                             |
| Title / Reason                       | Nome e motivo do bloqueio                                |
| IsActive                             | Status ativo ou inativo                                  |

---

### 6.2.1 Exemplo Prático: Bloqueio de Férias de Profissional

* Dra. Ana tira férias de **15/07 a 30/07**
* Horário: **Todos os dias úteis, 08:00 – 18:00**
* Tipo de exclusão: **ExclusionVisibility = SPECIALITY**
* Resultado:

  * Slots e HourBoxes do período ficam bloqueados
  * Nenhuma consulta pode ser agendada para Dra. Ana

---

### 6.2.2 Exemplo Prático: Manutenção de Equipamento

* Unidade de vacinação terá manutenção da sala **10:00 – 12:00**
* Data: 25/04/2024
* ExclusionVisibility: HEALTH_UNIT
* Resultado:

  * HourBoxes deste período bloqueados
  * Pacientes não conseguem marcar nesse horário

---

## 6.3 Regras de Aplicação

1. **Ordem de prioridade:**

   * ExcludeRange > ExcludeDay > Slot disponível
   * Ou seja, se há bloqueio de período, mesmo que o dia esteja livre, não é possível marcar

2. **Recorrência automática:**

   * ExcludeDay e ExcludeRange podem ser **recorrentes** (ex: toda segunda-feira, mensal, anual)
   * O sistema gera automaticamente os bloqueios futuros

3. **Visibilidade:**

   * Define **quem é afetado pelo bloqueio**:

     * SYSTEM: aplica globalmente
     * HEALTH_UNIT: aplica para toda a unidade
     * MEDICAL_AREA: aplica a uma especialidade médica
     * SPECIALITY: aplica apenas a agendas específicas

4. **Integração com Slots:**

   * Bloqueios não criam novas entidades de Slot
   * Eles **marcam slots existentes como indisponíveis** para os horários ou dias afetados

---

## 6.4 Benefício Prático

* Evita **agendamentos incorretos** em feriados, reuniões, férias ou manutenção
* Reduz **conflitos manuais**
* Permite **planejamento automatizado**, já que o sistema gera os bloqueios de acordo com a recorrência e visibilidade


---
