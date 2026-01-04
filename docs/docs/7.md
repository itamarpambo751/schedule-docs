
# 7. Horários de Trabalho dos Profissionais (WorkingHour)

O sistema permite que cada profissional tenha **horários de trabalho definidos por dia da semana, especialidade e unidade de saúde**, permitindo **planejar agendas e gerar slots automáticos** de forma precisa.

---

## 7.1 Estrutura de WorkingHour

| Campo                         | Descrição                                                                    |
| ----------------------------- | ---------------------------------------------------------------------------- |
| ProfessionalTaxId             | Identificador do profissional (CPF)                                          |
| HealthUnitTaxId               | Unidade de saúde onde atua                                                   |
| SpecialityCode                | Código da especialidade (ex: PED para Pediatria)                             |
| WeekDay                       | Dia da semana                                                                |
| StartAt / EndsAt              | Horário de início e fim do expediente                                        |
| ValidFrom / ValidTo           | Período de validade do horário (ex: contrato temporário, férias programadas) |
| TypeOfService                 | Tipo de atendimento: Presencial, Telemedicina ou Home                        |
| ExcludedDays / ExcludedRanges | Bloqueios aplicados ao profissional (feriados, folgas, férias)               |

---

## 7.2 Cadastro de Horário de Trabalho

### Exemplo Prático: Dra. Ana, Cardiologista

**Objetivo:** Configurar a rotina semanal de Dra. Ana na Unidade Hospitalar São Lucas.

| Dia     | Início | Fim   | Tipo de serviço |
| ------- | ------ | ----- | --------------- |
| Segunda | 13:00  | 18:00 | Presencial      |
| Terça   | 13:00  | 18:00 | Presencial      |
| Quarta  | 13:00  | 18:00 | Presencial      |
| Quinta  | 13:00  | 18:00 | Presencial      |
| Sexta   | 13:00  | 18:00 | Presencial      |
| Sábado  | 08:00  | 12:00 | Presencial      |
| Domingo | Folga  | -     | -               |

**Configuração no sistema:**

* Horários cadastrados no WorkingHour
* Dias e horas bloqueados automaticamente em caso de férias, feriados ou eventos

---

## 7.3 Integração com Slots

* Cada **Slot** representa um dia completo dentro de uma agenda (não um horário específico)
* Ao criar uma agenda com **recorrência e horário definido**, o sistema verifica:

  * Quais profissionais têm WorkingHour válidos naquele dia e horário
  * Se existem **Exclusões aplicadas ao profissional** (Ex: férias)
* O sistema **atribui automaticamente os profissionais disponíveis** aos slots gerados
* Caso múltiplos profissionais possam atender, o sistema pode:

  * Distribuir consultas de forma equilibrada
  * Permitir escolha manual (quando configurado)

---

## 7.4 Exemplo de Fluxo de Atribuição Automática

**Cenário:** Agenda de Pediatria, segunda-feira das 08:00 às 12:00

1. Sistema gera slot para 01/06/2024
2. Profissionais cadastrados com WorkingHour nesse horário:

   * Dr. Carlos (08:00-12:00)
   * Dra. Julia (08:00-12:00)
3. Sistema verifica exclusões:

   * Dr. Carlos está de folga → removido
   * Dra. Julia disponível → slot atribuído
4. Slot agora contém **Dra. Julia como profissional disponível**
5. Quando paciente marca consulta, sistema cria **HourBox** no horário escolhido dentro do slot

---

## 7.5 Bloqueios e Ajustes Dinâmicos

* Se um profissional adicionar uma folga ou sair de férias:

  * ExcludeDay ou ExcludeRange associado a ele
  * Slots afetados **perdem automaticamente o profissional**
  * Consultas existentes podem:

    * Ser remarcadas
    * Receber notificação de alteração

* Se um profissional altera horário:

  * O sistema atualiza automaticamente os slots futuros
  * Garantindo que **nenhum horário é atribuído fora da disponibilidade real**

---

## 7.6 Benefícios

* **Precisão:** Apenas profissionais disponíveis são considerados
* **Automação:** Não é necessário alocar manualmente cada consulta
* **Flexibilidade:** Horários podem variar por dia, tipo de serviço ou unidade
* **Integração total:** Slots, agendas, exclusões e consultas respeitam o horário definido


