# Estratégia de Tempo Real (.NET Core)

---

## 1) Canal Primário: **ASP.NET Core SignalR (WebSockets-first)**

**Porquê:** Integrado no .NET, simples de programar, com ótimo suporte a grupos, autenticação e *scale-out*.

**Fallbacks automáticos:** caso o WebSocket não esteja disponível, o SignalR alterna para **SSE (Server-Sent Events)** ou **Long Polling**, garantindo comunicação mesmo em ambientes restritos.

### Escala horizontal
- **Redis backplane** (para ambientes *on-premises* ou autogeridos).
- **Azure SignalR Service** (serviço gerido com *auto-scale* e menos manutenção).

### Eventos expostos
- `schedule.slots.generated` → quando um dia de *slots* é criado ou atualizado.  
- `schedule.slots.changed` → quando há *merge* aplicado ou o campo `isClosed` é alterado.  
- `appointment.created` / `appointment.status.changed` → criação ou mudança de estado de uma consulta.  
- `forwarding.created` / `forwarding.status.changed` → criação ou atualização de um encaminhamento.  
- `admin.job.progress` → progresso de tarefas administrativas (ex.: recálculo de capacidades, operações em massa, etc.).

### Tópicos / Grupos (granularidade)
Os grupos permitem envio seletivo sem desperdiçar tráfego, de acordo com o escopo do utilizador.

```
tenant:{healthUnitTaxId}
speciality:{specialityId}
schedule:{scheduleId}
professional:{professionalTaxId}
patient:{patientTaxId} // útil para notificações no portal do utente
```

➡️ **Regra:** ao conectar, o utilizador é automaticamente adicionado aos grupos correspondentes às suas *claims* (escopos).

---

## 2) Canal Interno (entre serviços)

Pode ser implementado com **gRPC streaming** ou **Kafka → Worker → SignalR**.

### Fluxo recomendado
1. Os eventos centrais são publicados no **Kafka** (ou RabbitMQ).  
2. Um *consumer* no **Worker Service** transforma os eventos em DTOs de notificação.  
3. O *Worker* envia os dados para o **Hub SignalR**, que os distribui aos clientes.

🎯 **Vantagem:** o Kafka serve como fonte de verdade e mantém a ordenação dos eventos, enquanto o SignalR apenas distribui as mensagens.

---

## 3) Push Notifications para Mobile

Utilize:
- **Firebase Cloud Messaging (FCM)** para Android e Web.
- **Apple Push Notification Service (APNs)** para iOS.

Esses canais servem para alertas discretos (ex.: “consulta alterada”), que ao serem clicados abrem o aplicativo — o qual, em seguida, conecta-se ao SignalR para sincronização detalhada.

🧠 **Importante:** Push é essencial quando o app está em *background*, pois os SOs limitam conexões WebSocket nesse estado.

---

## 4) Webhooks para integrações externas

- Parceiros externos que não utilizam *sockets* podem receber **webhooks assinados com HMAC**.  
- Implementar **reentrega automática** e **Dead Letter Queue (DLQ)** para garantir fiabilidade.

💡 Mantenha o Hub para UI e os Webhooks para sistemas externos.

---

## 5) Fiabilidade & Ordenação (essencial em sistemas de saúde)

- **Outbox Pattern:** o evento é gravado no banco (Postgres), depois publicado no Kafka.  
- **Idempotência:** garantida via `messageId` único.  
- **Sequenciamento:** ordenar eventos por entidade (`appointmentId`, `slotId`, etc.) para evitar inconsistências no cliente.  

### Rejoin com cursor
Cada mensagem contém um `sequence` incremental. O cliente armazena o último recebido (`lastSequence`).  
Ao reconectar, o cliente chama o endpoint REST com `?since=lastSequence` para realizar *catch-up* antes de reentrar no Hub.

### Backpressure
- Definir **limite de mensagens por segundo** por conexão.  
- **Coalescer eventos:** em rajadas, enviar mensagens agregadas (ex.: “slots changed (N updates)”) em vez de várias individuais.

---

## 6) Segurança

- **Autenticação JWT Bearer** no Hub (`AddAuthentication().AddJwtBearer()` + `MapHub`).  
- **Autorização por grupo:** validar *claims* antes de adicionar o utilizador.  
- **Proteção de PII:** nunca enviar dados sensíveis no socket — apenas IDs; o cliente busca os detalhes via REST se autorizado.  
- **Rate limiting / throttling** por conexão para evitar abuso.

---

## 7) Observabilidade

- **Métricas:** conexões ativas, mensagens/segundo, latência de entrega, tamanho das filas de saída.  
- **Logs:** *subscribe/unsubscribe*, quedas de conexão, erros de transporte.  
- **OpenTelemetry (OTel):** traçar o caminho completo “DB change → Kafka → Worker → SignalR → Cliente”.

---

## 8) Exemplo de Hub (simplificado)

```csharp
// Startup
app.MapHub<RealtimeHub>("/rt");

// Hub
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.SignalR;

[Authorize] // JWT
public class RealtimeHub : Hub
{
    public override async Task OnConnectedAsync()
    {
        var user = Context.User!;
        
        // Claims → associação a grupos (ex.: tenant e profissional)
        var tenant = user.FindFirst("healthUnitTaxId")?.Value;
        if (!string.IsNullOrEmpty(tenant))
            await Groups.AddToGroupAsync(Context.ConnectionId, $"tenant:{tenant}");

        var prof = user.FindFirst("professionalTaxId")?.Value;
        if (!string.IsNullOrEmpty(prof))
            await Groups.AddToGroupAsync(Context.ConnectionId, $"professional:{prof}");

        await Clients.Caller.SendAsync("connected", new { now = DateTime.UtcNow });
    }

    // Subscrição explícita a um schedule
    public Task SubscribeSchedule(string scheduleId)
        => Groups.AddToGroupAsync(Context.ConnectionId, $"schedule:{scheduleId}");

    public Task UnsubscribeSchedule(string scheduleId)
        => Groups.RemoveFromGroupAsync(Context.ConnectionId, $"schedule:{scheduleId}");
}
```

### Emissão a partir do Worker
```csharp
public class SlotNotifier
{
    private readonly IHubContext<RealtimeHub> _hub;

    public SlotNotifier(IHubContext<RealtimeHub> hub) => _hub = hub;

    public Task NotifySlotsChanged(string scheduleId, object payload)
        => _hub.Clients.Group($"schedule:{scheduleId}")
                       .SendAsync("schedule.slots.changed", payload);
}
```

---

## 9) Quando usar SSE (Server-Sent Events)

- Ideal para **dashboards read-only** ou **ambientes muito restritos**.  
- Suporta **reconexão automática** e **Last-Event-ID** nativo.  
- Comunicação **unidirecional (server → cliente)**.  

➡️ Mantenha o SignalR para casos **bidirecionais** (ex.: chats, confirmações, atualizações em tempo real).

---

## 10) Board de “quem recebe o quê”

| Destinatário | Tipos de Notificação |
|---------------|----------------------|
| Profissional | Status de consultas, convites e encaminhamentos |
| Paciente (Portal) | Confirmações, alterações ou cancelamentos |
| Admin / Secretaria | Mudanças de slots, sobrecargas e alertas |
| Especialidade | Picos de consultas, *no-show spikes*, estatísticas agregadas |
| Schedule | Ciclo completo dos slots — ideal para dashboards da agenda |

---

## TL;DR

- **SignalR (WebSockets)** → canal principal da UI.  
- **Kafka → Worker → Hub** → garante fiabilidade e escalabilidade.  
- **gRPC streaming** → comunicação entre microserviços.  
- **Webhooks / Push** → terceiros e apps mobile.  
- **Cursor / “since” / outbox / idempotência** → evita perda e desordem de mensagens.  
- **Segurança, grupos por tenant e observabilidade ponta a ponta.**
