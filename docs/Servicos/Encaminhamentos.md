# API Hospitalar — Encaminhamentos (Forwarding)

> Todas as datas em **UTC** (ISO 8601). Padrões: `page=1`, `limit=20`, `sortOrder=desc`.  
> Autenticação: Bearer Token. Autorização: RBAC/ABAC conforme perfis.

---

## 1. ENCAMINHAMENTOS (FORWARDING)

### 5.1. CRUD Básico
- **POST** `/forwardings` - Criar encaminhamento
- **GET** `/forwardings/{id}` - Obter encaminhamento
- **PUT** `/forwardings/{id}` - Atualizar encaminhamento
- **DELETE** `/forwardings/{id}` - Cancelar encaminhamento

### 5.2. Gestão de Estados
- **PATCH** `/forwardings/{id}/send` - Marcar como enviado
- **PATCH** `/forwardings/{id}/receive` - Marcar como recebido
- **PATCH** `/forwardings/{id}/accept` - Aceitar encaminhamento
- **PATCH** `/forwardings/{id}/reject` - Rejeitar encaminhamento
- **PATCH** `/forwardings/{id}/complete` - Completar encaminhamento

### 5.3. Consultas
- **GET** `/forwardings` - Listar encaminhamentos
- **GET** `/appointments/{appointmentId}/forwardings` - Encaminhamentos da marcação
- **GET** `/forwardings/status/{status}` - Encaminhamentos por status
- **GET** `/forwardings/receiver/{receiverTaxId}` - Encaminhamentos para profissional

---

# 🩺 Filtros para Listagem de Encaminhamentos (GET /forwardings)

## 🎯 FILTROS BÁSICOS E IDENTIFICAÇÃO

### 1.1. Identificação Direta
- **id** – ID específico do encaminhamento  
- **appointmentId** – Marcação específica  
- **senderTaxId** – Profissional que encaminhou  
- **receiverTaxId** – Profissional que recebeu  
- **medicalAreaId** – Área médica de destino  
- **specialityId** – Especialidade de destino  

---

## ⚖️ FILTROS DE STATUS E ESTADO

### 2.1. Status do Encaminhamento
- **status** – Status específico:  
  - PENDING  
  - SENT  
  - RECEIVED  
  - ACCEPTED  
  - REJECTED  
  - COMPLETED  
  - CANCELLED  

- **statusIn** – Múltiplos status (ex: `statusIn=PENDING,SENT`)  
- **isActive** – Encaminhamentos ativos (não cancelados/completados)  
- **isPending** – Aguardando ação (status: PENDING, SENT, RECEIVED)  
- **isCompleted** – Finalizados (status: COMPLETED, ACCEPTED)  
- **isRejected** – Rejeitados (status: REJECTED, CANCELLED)  

### 2.2. Estados de Fluxo
- **needsAction** – Requer ação (status: PENDING, SENT)  
- **awaitingResponse** – Aguardando resposta (status: SENT, RECEIVED)  
- **processed** – Processados (status: ACCEPTED, REJECTED, COMPLETED)  

---

## 📋 FILTROS DE DESTINO E PRIORIDADE

### 3.1. Destino do Encaminhamento
- **destination** – Tipo de destino:  
  - INTERNAL (Interno)  
  - EXTERNAL (Externo)  
- **hasReceiver** – Com destinatário definido (true/false)  
- **hasMedicalArea** – Com área médica definida (true/false)  
- **hasSpeciality** – Com especialidade definida (true/false)  

### 3.2. Prioridade
- **priority** – Nível de prioridade:  
  - NORMAL  
  - URGENT  
  - SERIOUS  
- **priorityIn** – Múltiplas prioridades  
- **isUrgent** – Prioritários (URGENT, SERIOUS)  

---

## 👥 FILTROS POR PARTICIPANTES

### 4.1. Remetente
- **senderTaxId** – Profissional que encaminhou  
- **senderTaxIds** – Múltiplos remetentes  

### 4.2. Destinatário
- **receiverTaxId** – Profissional que recebeu  
- **receiverTaxIds** – Múltiplos destinatários  
- **hasReceiver** – Com destinatário atribuído (true/false)  

### 4.3. Paciente (via Appointment)
- **patientTaxId** – Paciente da marcação  
- **patientAgeMin** – Idade mínima do paciente  
- **patientAgeMax** – Idade máxima do paciente  

---

## 📅 FILTROS TEMPORAIS

### 5.1. Períodos de Encaminhamento
- **forwardedAtFrom** – Encaminhados a partir de  
- **forwardedAtTo** – Encaminhados até  
- **forwardedDate** – Data específica de encaminhamento  
- **createdAtFrom** – Criados a partir de  
- **createdAtTo** – Criados até  
- **updatedAtFrom** – Atualizados a partir de  
- **updatedAtTo** – Atualizados até  

### 5.2. Períodos Especiais
- **today** – Encaminhados hoje  
- **thisWeek** – Encaminhados esta semana  
- **lastWeek** – Encaminhados na semana passada  
- **thisMonth** – Encaminhados este mês  
- **recent** – Encaminhamentos recentes (últimos 7 dias)  

---

## 🔗 FILTROS RELACIONADOS A MARCAÇÃO

### 6.1. Propriedades da Marcação
- **appointmentStatus** – Status da marcação relacionada  
- **appointmentTypeOfService** – Tipo de serviço da marcação  
- **appointmentProfessionalTaxId** – Profissional da marcação  
- **appointmentSelectedHourFrom** – Data/hora da marcação a partir de  
- **appointmentSelectedHourTo** – Data/hora da marcação até  

### 6.2. Informações da Agenda
- **healthUnitTaxId** – Unidade de saúde da marcação  
- **specialityId** – Especialidade da marcação  
- **typeOfSchedule** – Tipo de agenda da marcação  

---

## 📝 FILTROS DE CONTEÚDO

### 7.1. Motivo e Observações
- **reason** – Busca textual no motivo  
- **obs** – Busca textual nas observações  
- **hasReason** – Com motivo definido (true/false)  
- **hasObs** – Com observações (true/false)  
- **search** – Busca geral em motivo e observações  

### 7.2. Detalhes Específicos
- **reasonContains** – Contém termo específico no motivo  
- **obsContains** – Contém termo específico nas observações  

---

## 🎚️ FILTROS DE FLUXO DE TRABALHO

### 8.1. Encaminhamentos por Estágio
- **awaitingSending** – Aguardando envio (PENDING)  
- **awaitingReceipt** – Aguardando recebimento (SENT)  
- **awaitingDecision** – Aguardando decisão (RECEIVED)  
- **finalized** – Finalizados (ACCEPTED, REJECTED, COMPLETED)  

### 8.2. Alertas e Pendências
- **overdue** – Encaminhamentos em atraso (baseado em prioridade + tempo)  
- **highPriorityPending** – Prioritários pendentes (URGENT/SERIOUS + PENDING/SENT)  
- **requiresAttention** – Requerem atenção (status problemáticos + tempo)  

---

## 🔄 FILTROS DE PAGINAÇÃO E ORDENAÇÃO

### 9.1. Paginação
- **page** – Página atual (padrão: 1)  
- **limit** – Itens por página (padrão: 20, máximo: 100)  
- **offset** – Alternativa à paginação  

### 9.2. Ordenação
- **sortBy** – Campo para ordenação:  
  - forwardedAt  
  - createdAt  
  - updatedAt  
  - priority  
  - status  
  - senderTaxId  
  - receiverTaxId  
- **sortOrder** – `asc` ou `desc` (padrão: `desc` para datas)  

### 9.3. Seleção de Campos
- **fields** – Campos específicos a retornar  
- **include** – Relacionamentos a incluir:  
  - appointment  
  - appointment.slot  
  - appointment.schedule  
  - appointment.patient (dados básicos)  
  - sender (dados do profissional)  
  - receiver (dados do profissional)  

---

## 📊 EXEMPLOS PRÁTICOS DE USO

### Exemplo 1 – Encaminhamentos pendentes de um profissional
```http
GET /forwardings?senderTaxId=BI12345&status=PENDING&sortBy=forwardedAt&sortOrder=asc
```

### Exemplo 2 – Encaminhamentos para um destinatário
```http
GET /forwardings?receiverTaxId=BI67890&statusIn=SENT,RECEIVED&include=appointment,sender
```

### Exemplo 3 – Encaminhamentos urgentes
```http
GET /forwardings?priorityIn=URGENT,SERIOUS&statusIn=PENDING,SENT&forwardedAtFrom=2024-12-01
```

### Exemplo 4 – Encaminhamentos externos
```http
GET /forwardings?destination=EXTERNAL&status=COMPLETED&forwardedAtFrom=2024-11-01&include=appointment
```

### Exemplo 5 – Encaminhamentos por especialidade
```http
GET /forwardings?specialityId=CARDIOLOGY&hasReceiver=true&status=ACCEPTED&forwardedAtFrom=2024-12-01
```

### Exemplo 6 – Encaminhamentos de um paciente
```http
GET /forwardings?patientTaxId=BI99999&include=appointment,sender,receiver&sortBy=forwardedAt&sortOrder=desc
```

### Exemplo 7 – Encaminhamentos recentes
```http
GET /forwardings?recent=true&statusIn=PENDING,SENT,RECEIVED&include=appointment.schedule
```

### Exemplo 8 – Estatísticas de encaminhamento
```http
GET /forwardings?healthUnitTaxId=540102938&forwardedAtFrom=2024-12-01&forwardedAtTo=2024-12-31&fields=id,status,priority,forwardedAt
```

### Exemplo 9 – Encaminhamentos com observações
```http
GET /forwardings?hasObs=true&status=COMPLETED&forwardedAtFrom=2024-11-01&obsContains=avaliação
```

### Exemplo 10 – Encaminhamentos rejeitados/cancelados
```http
GET /forwardings?statusIn=REJECTED,CANCELLED&forwardedAtFrom=2024-12-01&include=appointment,sender,receiver
```

### Exemplo 11 – Encaminhamentos por motivo específico
```http
GET /forwardings?reasonContains=consulta&priority=URGENT&status=PENDING
```

### Exemplo 12 – Dashboard de encaminhamentos
```http
GET /forwardings?senderTaxId=BI12345&forwardedAtFrom=2024-12-01&include=appointment.patient,receiver&sortBy=priority&sortOrder=desc&sortBy=forwardedAt&sortOrder=asc
```

### Exemplo 13 – Encaminhamentos internos pendentes
```http
GET /forwardings?destination=INTERNAL&status=PENDING&hasReceiver=true&forwardedAtFrom=2024-12-01
```

### Exemplo 14 – Encaminhamentos por área médica
```http
GET /forwardings?medicalAreaId=ONCOLOGY&statusIn=ACCEPTED,COMPLETED&forwardedAtFrom=2024-11-01
```

### Exemplo 15 – Busca complexa com múltiplos filtros
```http
GET /forwardings?healthUnitTaxId=540102938&specialityId=NEUROLOGY&destination=INTERNAL&priority=URGENT&statusIn=PENDING,SENT&forwardedAtFrom=2024-12-01&hasReceiver=true&include=appointment.schedule,sender,receiver&sortBy=priority&sortOrder=desc&sortBy=forwardedAt&sortOrder=asc
```

---

## ⚙️ ESTRUTURA DE RESPOSTA
### Resposta Paginada com Estatísticas

```json
{
  "data": [
    {
      "id": "fwd-123",
      "appointmentId": "appt-456",
      "destination": "INTERNAL",
      "medicalAreaId": "CARDIOLOGY",
      "specialityId": "CARDIOLOGY",
      "receiverTaxId": "BI67890",
      "senderTaxId": "BI12345",
      "reason": "Encaminhamento para avaliação cardiológica",
      "priority": "URGENT",
      "status": "SENT",
      "forwardedAt": "2024-12-15T10:00:00Z",
      "obs": "Paciente com histórico familiar",
      "createdAt": "2024-12-15T09:30:00Z",
      "updatedAt": "2024-12-15T10:00:00Z",
      "appointment": {
        "id": "appt-456",
        "selectedHour": "2024-12-14T14:00:00Z",
        "patientTaxId": "BI99999",
        "patientAge": 45,
        "professionalTaxId": "BI12345",
        "typeOfService": "IN_PERSON",
        "status": "COMPLETED"
      },
      "sender": { "name": "Dr. Silva", "taxId": "BI12345" },
      "receiver": { "name": "Dr. Santos", "taxId": "BI67890" }
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 85,
    "totalPages": 5,
    "hasNext": true,
    "hasPrev": false
  },
  "stats": {
    "totalForwardings": 85,
    "byStatus": {
      "PENDING": 15,
      "SENT": 20,
      "RECEIVED": 10,
      "ACCEPTED": 25,
      "REJECTED": 5,
      "COMPLETED": 10
    },
    "byPriority": {
      "NORMAL": 40,
      "URGENT": 35,
      "SERIOUS": 10
    },
    "byDestination": {
      "INTERNAL": 60,
      "EXTERNAL": 25
    }
  }
}
```
