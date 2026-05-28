# Evolution API v2 — Webhooks

> Fonte: [doc.evolution-api.com/v2/en/configuration/webhooks](https://doc.evolution-api.com/v2/en/configuration/webhooks)

---

## Configurar Webhook para uma Instância

```
POST /webhook/set/{instance}
```

**Headers:**
```
Content-Type: application/json
apikey: SUA_API_KEY
```

**Body:**
```json
{
  "url": "https://meu-servidor.com/webhook/whatsapp",
  "enabled": true,
  "webhookByEvents": false,
  "webhookBase64": false,
  "events": [
    "MESSAGES_UPSERT",
    "MESSAGES_UPDATE",
    "CONNECTION_UPDATE",
    "SEND_MESSAGE"
  ]
}
```

**Resposta (201):**
```json
{
  "webhook": {
    "instanceName": "pv360",
    "webhook": {
      "url": "https://meu-servidor.com/webhook/whatsapp",
      "events": ["MESSAGES_UPSERT", "MESSAGES_UPDATE", "CONNECTION_UPDATE", "SEND_MESSAGE"],
      "enabled": true
    }
  }
}
```

---

## Ver Webhook Configurado

```
GET /webhook/find/{instance}
```

```bash
curl http://72.61.48.156:8080/webhook/find/pv360 \
  -H "apikey: suporte123"
```

---

## Todos os Eventos Disponíveis

| Evento | Quando dispara |
|--------|---------------|
| `APPLICATION_STARTUP` | Evolution API inicia |
| `QRCODE_UPDATED` | Novo QR Code gerado |
| `CONNECTION_UPDATE` | Status da conexão WhatsApp muda |
| `NEW_TOKEN` | JWT renovado |
| `MESSAGES_SET` | Histórico inicial de mensagens |
| `MESSAGES_UPSERT` | **Nova mensagem recebida ou enviada** ← principal |
| `MESSAGES_UPDATE` | Mensagem atualizada (status, reação, etc.) |
| `MESSAGES_DELETE` | Mensagem deletada |
| `SEND_MESSAGE` | Mensagem enviada via API |
| `CONTACTS_SET` | Lista inicial de contatos |
| `CONTACTS_UPSERT` | Novo contato ou atualização |
| `CONTACTS_UPDATE` | Contato atualizado |
| `PRESENCE_UPDATE` | Online/offline/digitando |
| `CHATS_SET` | Lista inicial de chats |
| `CHATS_UPSERT` | Novo chat |
| `CHATS_UPDATE` | Chat atualizado |
| `CHATS_DELETE` | Chat deletado |
| `GROUPS_UPSERT` | Grupo criado ou atualizado |
| `GROUP_UPDATE` | Grupo modificado |
| `GROUP_PARTICIPANTS_UPDATE` | Participante adicionado/removido |
| `CALL` | Chamada recebida |
| `TYPEBOT_START` | Bot Typebot iniciado |
| `TYPEBOT_CHANGE_STATUS` | Status do Typebot mudou |

---

## Payload MESSAGES_UPSERT

```json
{
  "event": "messages.upsert",
  "instance": "pv360",
  "data": {
    "key": {
      "remoteJid": "5511997663780@s.whatsapp.net",
      "fromMe": false,
      "id": "3EB0EF12345678"
    },
    "pushName": "João Cliente",
    "status": "DELIVERY_ACK",
    "message": {
      "conversation": "Olá, preciso de ajuda com meu pedido"
    },
    "messageType": "conversation",
    "messageTimestamp": 1717689097,
    "instanceId": "abc123",
    "source": "android"
  },
  "destination": "https://meu-servidor.com/webhook/whatsapp",
  "date_time": "2026-05-28T14:30:00.000Z",
  "sender": "5511997663780@s.whatsapp.net",
  "server_url": "http://72.61.48.156:8080",
  "apikey": "suporte123"
}
```

---

## webhookByEvents — Roteamento por Evento

Se `webhookByEvents: true`, o evento é **appended na URL** com hífen:

```
MESSAGES_UPSERT   → https://meu-servidor.com/webhook/messages-upsert
CONNECTION_UPDATE → https://meu-servidor.com/webhook/connection-update
QRCODE_UPDATED    → https://meu-servidor.com/webhook/qrcode-updated
```

Útil para roteamento sem switch/case no servidor.

---

## Webhook Global (via .env)

Para receber todos os eventos de todas as instâncias num único endpoint:

```bash
# .env
WEBHOOK_GLOBAL_ENABLED=true
WEBHOOK_GLOBAL_URL=https://meu-servidor.com/webhook/global
WEBHOOK_GLOBAL_WEBHOOK_BY_EVENTS=false
WEBHOOK_EVENTS_MESSAGES_UPSERT=true
WEBHOOK_EVENTS_CONNECTION_UPDATE=true
# ... outros eventos
```

---

## Configuração atual da instância pv360 (VP)

```bash
# Ver configuração atual
curl http://72.61.48.156:8080/webhook/find/pv360 \
  -H "apikey: suporte123"
```

```json
{
  "url": "https://posvenda360.vpsistema.com/api/whatsapp/webhook",
  "enabled": true,
  "webhookByEvents": false,
  "events": [
    "MESSAGES_UPSERT",
    "MESSAGES_UPDATE",
    "SEND_MESSAGE",
    "CONNECTION_UPDATE"
  ]
}
```

---

## Handler do Webhook no pv360

O arquivo `nodejs/hostinger/server.mjs` intercepta `POST /api/whatsapp/webhook` ANTES do TanStack Start.

```javascript
// Fluxo simplificado em server.mjs
app.post("/api/whatsapp/webhook", async (req, res) => {
  const { event, data } = req.body;

  if (event === "messages.upsert") {
    const { remoteJid, fromMe, message } = data;
    
    if (fromMe) return;           // Ignorar próprias mensagens
    if (isGroup(remoteJid)) return; // Ignorar grupos
    
    // Salvar no Supabase
    await saveToSupabase(data);
    
    // Auto-reply via Claude (se HERMES_AUTO_REPLY=true)
    if (process.env.HERMES_AUTO_REPLY === "true") {
      await automateIncoming(remoteJid);
    }
  }
  
  res.json({ status: "ok" });
});
```

> ⚠️ A rota `/api/webhook/evolution` no TanStack **não é atingida** no deploy Hostinger.
> O handler ativo é **apenas** o `server.mjs`.
