
# 5. Marcação de Consultas (Appointments)

O **Appointment** é a entidade central para registrar que **um paciente marcou uma consulta**. Ele integra informações do **slot/dia**, **HourBox/hora específica**, **profissional**, **unidade de saúde**, **tipo de serviço**, e **pagamento**.

---

## 5.1 Estrutura de Appointment

Principais campos e suas funções:

| Campo                                        | Descrição                                                |
| -------------------------------------------- | -------------------------------------------------------- |
| SlotId                                       | Identifica o **dia** da consulta                         |
| SelectedHour                                 | Hora específica (HourBox) escolhida                      |
| ProfessionalTaxId                            | Médico/profissional responsável                          |
| PatientTaxId                                 | CPF do paciente                                          |
| PatientAge / PatientGender                   | Validação automática de idade e gênero                   |
| HealthUnitTaxId                              | Unidade de saúde onde a consulta será realizada          |
| Status                                       | Estado da consulta (CONFIRMED, CANCELLED, NO_SHOW, etc.) |
| TypeOfService                                | Presencial, telemedicina ou home care                    |
| RoomLink                                     | Link da sala virtual (para telemedicina)                 |
| PaymentId                                    | Associa pagamento quando necessário                      |
| IsCancelled / CancelledAt / CancelledByTaxId | Controle de cancelamentos                                |
| HasReschedule / WasRescheduled               | Controle de remarcações                                  |

---

## 5.2 Fluxo de Agendamento

### Passo 1: Paciente escolhe especialidade e unidade

- Exemplo: Maria quer pediatria no **Hospital São Lucas**.
- Sistema retorna **dias com slots disponíveis** e **horários (HourBoxes)**.

### Passo 2: Seleção do slot e hora

1. Paciente escolhe o **Slot** (dia) desejado
2. Sistema lista **HourBoxes disponíveis**:

   - **AVAILABLE:** pode marcar
   - **BOOKED / HELD / BLOCKED / CANCELLED:** não disponível

**Exemplo de retorno:**

| Hora  | Status    | Profissional disponível |
| ----- | --------- | ----------------------- |
| 08:00 | AVAILABLE | Dr. Carlos              |
| 08:30 | AVAILABLE | Dr. Carlos              |
| 09:00 | BOOKED    | Dr. Carlos              |
| 09:30 | BLOCKED   | Nenhum                  |

---

### Passo 3: Validação de regras automáticas

Antes de confirmar, o sistema verifica:

- **Idade do paciente:** compatível com especialidade (ex: pediatria ≤18 anos)
- **Gênero do paciente:** se houver restrição (ex: ginecologia só feminino)
- **Horário disponível:** HourBox deve estar AVAILABLE
- **Pagamento pendente:** se consulta exigir pagamento antecipado
- **Limite de cancelamento/remarcação:** aplica políticas de tempo mínimo

> Caso alguma regra falhe, o sistema retorna mensagem clara para o paciente.

---

### Passo 4: Confirmação do agendamento

1. Sistema cria o **Appointment** vinculado ao Slot e HourBox
2. Atualiza **HourBox.Status → BOOKED**
3. Se telemedicina, gera automaticamente **RoomLink**
4. Notifica paciente e profissional via SMS/email

**Exemplo:**

- Paciente: João Souza, 5 anos
- Especialidade: Pediatria
- Data: 20/04/2024
- Hora: 08:00
- Profissional: Dr. Carlos

**Resultado no sistema:**

- Appointment.Status = CONFIRMED
- HourBox.Status = BOOKED
- RoomLink = null (presencial)

---

## 5.3 Regras de Cancelamento e Remarcação

### 5.3.1 Cancelamento

- **Prazo mínimo:** definido na Schedule (ex: 24h antes)
- **Ação:** Atualiza Appointment.IsCancelled = true, HourBox.Status → AVAILABLE
- **Multa:** se cancelamento tardio, registra penalidade ou cobrança automática

### 5.3.2 Remarcação

- Paciente solicita remarcação
- Sistema verifica:
  - Se já houve remarcação anterior (HasReschedule / WasRescheduled)
  - Disponibilidade do novo slot/hora
  - Limite de prazo para remarcação
- Se permitido:
  - Novo Appointment é criado, antigo marcado como remarcado
  - HourBoxes atualizados: antigo → AVAILABLE, novo → BOOKED

---

## 5.4 Exemplo Prático Completo

### Cenário: Consulta de Telemedicina

- Paciente: Ana Lima
- Especialidade: Cardiologia
- Data: 22/04/2024
- Hora: 14:00
- Tipo de serviço: Telemedicina
- Profissional: Dr. Pedro

**Passos no sistema:**

1. Sistema verifica Slot (22/04) e HourBox (14:00)
2. Confirma que Dr. Pedro está disponível (WorkingHour)
3. Gera **RoomLink:** https://meet.hospital/xyz123
4. Marca Appointment.Status = CONFIRMED
5. Notifica paciente e Dr. Pedro
6. 15 minutos antes, envia SMS com RoomLink

**Se paciente tentar marcar fora do horário:**

- Horário bloqueado ou indisponível → mensagem de erro
- Sugere próximos horários disponíveis

---

## 5.5 Considerações importantes

- **Appointment é dependente do Slot e HourBox:** sem dia e hora válidos, não é possível marcar
- **Validações automáticas** reduzem erros de agendamento
- **Integração com WorkingHour** garante que apenas profissionais disponíveis sejam atribuídos
- **Telemedicina e RoomLink** são gerados automaticamente, sem necessidade de intervenção manual
- **Exclusões e bloqueios** aplicados antes do agendamento garantem consistência e evitam conflitos
