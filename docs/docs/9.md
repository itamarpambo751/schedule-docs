
# 9. Marcação de Consultas (Appointments)

O **Appointment** é a entidade central que conecta **paciente, slot, profissional e regras da agenda**. Cada agendamento representa **uma consulta real** que será realizada presencialmente, por telemedicina ou em domicílio.

---

## 9.1 Processo de Marcação

### Passo 1: Paciente seleciona especialidade e unidade

* Paciente acessa o sistema (web ou app)
* Escolhe:

  * Especialidade (ex: Pediatria)
  * Unidade de saúde (ex: Hospital São Lucas)
* Sistema filtra **slots disponíveis** com base:

  * Tipo de serviço (presencial, telemedicina, home care)
  * Disponibilidade de profissionais
  * Excluídos por feriados/férias (ExcludeDay/ExcludeRange)
  * Limites de capacidade e horário

---

### Passo 2: Escolha de Slot e HourBox

* Sistema mostra **slots de acordo com a agenda**
* Para cada slot, apresenta **hourboxes** disponíveis

  * Indica número de vagas restantes
  * Mostra se há restrições de gênero ou idade
* Paciente seleciona **hourbox** desejada

  * Exemplo: 08:30 do dia 01/06/2024

---

### Passo 3: Preenchimento de dados do paciente

* Nome, idade, gênero, CPF ou outro TaxId
* Telefone de contato
* Informações adicionais (dependendo do tipo de consulta)
* Sistema valida automaticamente:

  * **Idade compatível com a especialidade**
  * **Gênero permitido** (ex: ginecologia somente mulheres)
  * **Limite de vagas do hourbox**
  * **Pagamento necessário** (PaymentId)

---

### Passo 4: Confirmação e bloqueio do hourbox

* Ao confirmar:

  * HourBox passa para **BOOKED**
  * Consulta ganha status **PENDING** ou **CONFIRMED**, dependendo de pagamento
  * Sistema registra:

    * Horário e slot
    * Profissional atribuído
    * Tipo de serviço
    * Data e hora de criação
* Notificações automáticas enviadas para paciente e profissional

---

## 9.2 Tipos de Status de Consulta

| Status                    | Significado                                       |
| ------------------------- | ------------------------------------------------- |
| PENDING                   | Consulta criada, aguardando confirmação/pagamento |
| PENDING_FOR_PAYMENT       | Consulta aguardando pagamento                     |
| PAYMENT_EXPIRED           | Pagamento não realizado dentro do prazo           |
| CONFIRMED                 | Consulta confirmada                               |
| CHECKED_IN                | Paciente chegou para consulta                     |
| IN_SESSION                | Consulta em andamento                             |
| COMPLETED                 | Consulta realizada                                |
| CANCELLED_BY_PATIENT      | Cancelamento pelo paciente                        |
| CANCELLED_BY_PROFESSIONAL | Cancelamento pelo profissional                    |
| CANCELLED_BY_SYSTEM       | Cancelamento automático por regras do sistema     |
| NO_SHOW                   | Paciente não compareceu                           |
| FOLLOW_UP_REQUIRED        | Necessário retorno ou acompanhamento              |
| DOCUMENT_PENDING          | Documentos ou exames pendentes                    |
| EXPIRED                   | Consulta expirada sem ação                        |

---

## 9.3 Regras Automáticas

* **Idade e gênero:** validação automática com base na agenda do slot
* **Limite de vagas:** não permite ultrapassar capacidade
* **Cancelamento:** apenas permitido se respeitar **advanceCancellationInHours** da agenda
* **Remarcação:** controla se **HasReschedule** ou **WasRescheduled** já foram usados
* **Pagamento:** consulta só passa para CONFIRMED após pagamento, se necessário
* **Telemedicina:** sistema gera automaticamente **roomLink** e código de acesso

---

## 9.4 Exemplos Práticos

### 9.4.1 Consulta Presencial

* Paciente: João, 5 anos
* Agenda: Pediatria, Dr. Carlos
* Slot: 01/06/2024
* HourBox: 08:30 (2 vagas)
* Processo:

  1. João seleciona 08:30
  2. Sistema valida idade e gênero
  3. Confirmação → status **CONFIRMED**
  4. HourBox 08:30 agora tem 1 vaga restante

---

### 9.4.2 Consulta Telemedicina

* Paciente: Maria, adulta
* Agenda: Psicologia, Dra. Ana
* Tipo de serviço: TELEMEDICINE
* Slot: 03/06/2024
* HourBox: 10:00
* Sistema gera:

  * Link da sala virtual: `https://meet.hospital/abc123`
  * Código de acesso: `654321`
* Notificações enviadas 15 minutos antes

---

### 9.4.3 Cancelamento automático por regras

* Paciente tenta cancelar fora do prazo (menos que **advanceCancellationInHours**)
* Sistema aplica regra:

  * HourBox volta a **AVAILABLE** se cancelamento permitido
  * Caso contrário, marca **NO_SHOW** ou aplica multa

---

