
# 8. Slots e HourBoxes

No sistema, **Slot** e **HourBox** são conceitos distintos, mas interligados:

* **Slot:** representa **um dia inteiro dentro de uma agenda**, incluindo informações sobre dia da semana, data específica, profissionais disponíveis e regras de agendamento.
* **HourBox:** representa **cada horário específico disponível dentro do Slot**, onde os pacientes podem marcar consultas.

Essa separação permite **flexibilidade**, como definir diferentes durações de consultas, capacidade por horário e tipos de serviço.

---

## 8.1 Tipos de Slots

Existem dois modos principais de operação de slot:

| Tipo                      | Descrição                                                                         | Exemplo                                                                                                  |
| ------------------------- | --------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| **Timed (TEMPO FIXO)**    | Cada HourBox representa um intervalo de tempo fixo (ex: 30 minutos).              | Agenda de Pediatria das 08:00 às 12:00 com duração de 30 minutos → gera HourBoxes: 08:00, 08:30, 09:00 … |
| **Capacity (CAPACIDADE)** | Cada horário tem uma capacidade máxima de pacientes, independente do tempo exato. | Agenda de vacinação com 2 vagas por horário: 08:00 (2 vagas), 08:30 (2 vagas)                            |

> Slots podem ser **gerados automaticamente** pela agenda ou criados manualmente se necessário.

---

## 8.2 Geração Automática de Slots

Ao criar uma agenda:

1. Definir **recorrência**: diária, semanal, mensal, personalizada
2. Informar **dias da semana e horários** de atendimento
3. Sistema verifica **horários de trabalho dos profissionais** (WorkingHour)
4. Para cada dia dentro do período da agenda:

   * Cria **um Slot**
   * Atribui automaticamente **profissionais disponíveis**
   * Cria **HourBoxes** para cada intervalo de tempo, com base no tipo de slot (Timed ou Capacity)
   * Considera **exclusões** (feriados, férias, folgas)

**Exemplo Prático:**

Agenda de Pediatria (Dr. Carlos)

* Segunda a sexta, 08:00-12:00
* Duração: 30 minutos
* Capacidade: 2 pacientes por horário

Sistema gera:

| Slot (Data) | HourBoxes                                           |
| ----------- | --------------------------------------------------- |
| 01/06/2024  | 08:00 (2 vagas), 08:30 (2 vagas), … 11:30 (2 vagas) |
| 02/06/2024  | 08:00 (2 vagas), 08:30 (2 vagas), … 11:30 (2 vagas) |
| …           | …                                                   |

> Se Dr. Carlos estiver de férias no dia 02/06/2024, slot desse dia **não terá HourBoxes disponíveis**.

---

## 8.3 Exclusões e Bloqueios

### 8.3.1 ExcludeDay

* Bloqueia **um dia inteiro**
* Pode ser aplicado ao Slot ou a agendas específicas
* Exemplo: 07/09/2024 é feriado → nenhum HourBox disponível nesse Slot

### 8.3.2 ExcludeRange

* Bloqueia **intervalos dentro de um dia**
* Exemplo: reunião da equipe segunda-feira 14:00-15:00
* Sistema ajusta os HourBoxes afetados → não podem ser marcados

> Ambos podem ter **recorrência** (semanal, mensal, anual) e **visibilidade** (Sistema, Unidade, Especialidade)

---

## 8.4 Atribuição Automática de Profissionais

* Ao gerar slots, sistema verifica **WorkingHours**
* Se mais de um profissional disponível:

  * Pode distribuir pacientes de forma balanceada
  * Ou permitir seleção manual pelo paciente (dependendo da configuração da agenda)
* Se o profissional estiver bloqueado (Ex: folga), ele é **removido do Slot** automaticamente

---

## 8.5 Benefícios do Sistema de Slots

* **Automatização total:** não precisa criar cada horário manualmente
* **Flexibilidade de capacidade:** permite desde horários individuais até grandes grupos (vacinas, exames)
* **Integração com exclusões:** feriados, férias e reuniões ajustam os horários automaticamente
* **Segurança e previsibilidade:** só profissionais disponíveis são atribuídos


---

