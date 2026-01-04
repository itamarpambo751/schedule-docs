
# 11. Horários de Trabalho de Profissionais (WorkingHours)

O sistema permite registrar **horários de trabalho individuais para cada profissional de saúde**. Essa configuração é essencial para que a **agenda funcione corretamente**, garantindo que os slots só sejam criados em horários em que o profissional está disponível.

---

## 11.1 Conceito

* Cada profissional pode ter **vários horários de trabalho por semana**, em diferentes dias e horários.
* Cada horário é vinculado a uma **unidade de saúde** e a uma **especialidade**.
* É possível registrar **períodos de validade**, caso o profissional só trabalhe temporariamente ou em datas específicas.
* Horários podem ter **exclusões** (feriados, férias, reuniões).

**Exemplo:**

* Dra. Ana, cardiologista:

  * Segunda a sexta: 13:00 às 18:00
  * Sábado: 08:00 às 12:00
  * Domingo: folga

---

## 11.2 Campos importantes

| Campo                         | Função                                                      |
| ----------------------------- | ----------------------------------------------------------- |
| ProfessionalTaxId             | Identificador do profissional (CPF)                         |
| HealthUnitTaxId               | Unidade de saúde onde atua                                  |
| SpecialityCode                | Código da especialidade (ex: CARD para Cardiologia)         |
| WeekDay                       | Dia da semana                                               |
| StartAt / EndsAt              | Horário de início e fim                                     |
| ValidFrom / ValidTo           | Período de validade do horário (opcional)                   |
| TypeOfService                 | Tipo de atendimento: presencial, telemedicina ou domiciliar |
| ExcludedDays / ExcludedRanges | Dias e horários bloqueados que afetam esse horário          |

---

## 11.3 Exemplo prático

**Cenário:** Dr. João é pediatra e trabalha em duas unidades de saúde com horários distintos.

| Unidade            | Dia       | Horário       |
| ------------------ | --------- | ------------- |
| Hospital São Lucas | Seg a Sex | 08:00 – 12:00 |
| Clínica São José   | Seg a Sex | 14:00 – 18:00 |

**Configuração no sistema:**

1. Criar **WorkingHour** para cada combinação de unidade e dia
2. Definir `StartAt`, `EndsAt`, `WeekDay`
3. Definir `SpecialityCode = PED`
4. Adicionar exclusões se necessário (feriados, férias)
5. Salvar

**Resultado:** Sistema sabe exatamente **quando Dr. João está disponível** e só gera slots nesses horários.

---

## 11.4 Relação com Slots e Agendas

* Durante a **geração automática de slots** em uma agenda:

  1. Sistema verifica todos os horários de trabalho ativos dos profissionais disponíveis
  2. Apenas cria **slots em dias e horários compatíveis**
  3. Atribui automaticamente os profissionais aos slots, respeitando:

     * Dias da semana
     * Horários
     * Tipo de serviço
     * Exceções de exclusão
* Se uma agenda tiver `EachSlotInheritAllScheduleProfessionals = true`, cada slot herda todos os profissionais compatíveis para aquele dia/hora.

---

## 11.5 Exemplos de exclusões

* **Férias**: Dr. João tira férias de 10 a 20/07 → horário marcado como **excluído** no WorkingHour
* **Reunião**: Segunda-feira, 15:00 às 16:00 → intervalo bloqueado
* **Manutenção**: Sábado, 08:00 às 09:00 → bloqueio parcial

O sistema **calcula automaticamente** que não haverá slots disponíveis nesses horários.

---

## 11.6 Benefícios

* Garante que **nenhum slot seja criado fora da disponibilidade real do profissional**
* Permite **atribuição automática** de profissionais aos slots
* Evita marcações incorretas, cancelamentos ou conflitos de agenda
* Facilita gestão de múltiplos profissionais em várias unidades e especialidades
