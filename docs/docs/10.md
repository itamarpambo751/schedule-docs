
# 10. Exclusões e Bloqueios Avançados (ExcludeDay e ExcludeRange)

O sistema permite que **dias ou períodos inteiros sejam bloqueados** para agendamento. Isso garante que **consultas não possam ser marcadas em datas ou horários indisponíveis**, seja por feriado, manutenção, reunião ou indisponibilidade de profissionais.

Existem **dois tipos principais de exclusão**:

1. **ExcludeDay** → Bloqueia dias inteiros ou padrões de dias da semana
2. **ExcludeRange** → Bloqueia apenas determinados intervalos de horário dentro de dias específicos ou recorrentes

---

## 10.1 ExcludeDay – Bloqueio de Dias Inteiros

### 10.1.1 O que é

* Representa **um dia ou conjunto de dias da semana** que deve ser excluído do agendamento.
* Pode ser **específico** (07/09/2024) ou **recorrente** (todas as segundas-feiras).

### 10.1.2 Campos importantes

| Campo               | Função                                                          |
| ------------------- | --------------------------------------------------------------- |
| SpecificDate        | Data exata a ser bloqueada (opcional)                           |
| WeekDays            | Lista de dias da semana a bloquear (segunda, terça…)            |
| TypeOfRecurrence    | Recorrência do bloqueio (Diário, Semanal, Mensal…)              |
| ExclusionVisibility | Onde o bloqueio será aplicado (Sistema, Unidade, Especialidade) |
| Title / Reason      | Nome e motivo da exclusão (ex: “Feriado Nacional”)              |
| DefinedBy           | Quem definiu o bloqueio                                         |

### 10.1.3 Exemplo prático

**Cenário:** Hospital terá feriado em 07/09/2024.

**Passos no sistema:**

1. Criar novo **ExcludeDay**
2. Definir `SpecificDate = 07/09/2024`
3. `ExclusionVisibility = HEALTH_UNIT`
4. `Title = "Feriado Nacional"`
5. `Reason = "Independência do Brasil"`
6. Salvar

**Resultado:** Nenhum slot será gerado para 07/09/2024, impossibilitando marcação de consultas.

---

### 10.1.4 Exemplo recorrente

**Cenário:** Toda segunda-feira, equipe tem reunião das 14h às 15h.

* Criar **ExcludeDay** com `WeekDays = MONDAY`
* `TypeOfRecurrence = WEEKLY`
* `ExclusionVisibility = HEALTH_UNIT`
* Sistema aplicará automaticamente essa exclusão **em todas as segundas-feiras futuras**, ajustando slots gerados.

---

## 10.2 ExcludeRange – Bloqueio de Intervalos de Horário

### 10.2.1 O que é

* Bloqueia **somente horários específicos** dentro de um dia ou slot.
* Ideal para feriados parciais, reuniões ou horários de manutenção de equipamentos.

### 10.2.2 Campos importantes

| Campo                   | Função                                                   |
| ----------------------- | -------------------------------------------------------- |
| StartTime / EndTime     | Intervalo de horário a bloquear                          |
| StartDate / EndDate     | Período em que a exclusão vale                           |
| ExcludeForAllSlots      | Bloqueia todos os slots do dia ou apenas os selecionados |
| ExcludeFor              | Lista de dias da semana aplicáveis                       |
| ExcludeForSpecificDates | Lista de datas específicas                               |
| TypeOfRecurrence        | Recorrência do bloqueio (semanal, mensal, etc.)          |
| ExclusionVisibility     | Onde o bloqueio será aplicado                            |
| Title / Reason          | Nome e motivo do bloqueio                                |

---

### 10.2.3 Exemplo prático

**Cenário:** Dra. Ana vai tirar férias de 15 a 30 de julho, mas só nas tardes.

* Criar **ExcludeRange**
* `StartDate = 15/07/2024`, `EndDate = 30/07/2024`
* `StartTime = 13:00`, `EndTime = 18:00`
* `ExclusionVisibility = HEALTH_UNIT`
* `Reason = "Férias Dra. Ana"`

**Resultado:**

* Slots gerados para essas datas terão **hourboxes bloqueadas entre 13:00 e 18:00**
* Pacientes não poderão marcar consultas nesse período

---

### 10.2.4 Recorrências e impacto

* **Diário:** bloqueia todos os dias dentro do período
* **Semanal:** bloqueia determinados dias da semana
* **Mensal:** bloqueia datas específicas de cada mês
* **Anual:** bloqueia feriados ou datas fixas todos os anos

O sistema **aplica essas regras durante a geração automática de slots**, garantindo que nenhum hourbox inválido seja criado.

---

### 10.3 Integração com Slots e Agendas

* Antes de gerar hourboxes, o sistema **verifica todas as exclusões aplicáveis** (ExcludeDay e ExcludeRange)
* Slots em dias totalmente bloqueados não são criados
* Para ranges parciais, apenas hourboxes dentro do intervalo são bloqueadas
* Atribuição de profissionais aos slots também respeita as exclusões


---

