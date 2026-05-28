# n8n — Webhook Nodes

> Fonte: [docs.n8n.io](https://docs.n8n.io/)

O Webhook node permite que n8n receba requisições HTTP externas e use como trigger de workflows — ideal para integrar com Evolution API.

---

## Configuração básica

1. Adicionar **Webhook node** ao workflow
2. Configurar:
   - **HTTP Method:** POST (para webhooks de entrada)
   - **Path:** `/whatsapp` (fica disponível em `https://n8n.host/webhook/whatsapp`)
   - **Authentication:** Header Auth, Basic Auth, ou None
   - **Response Mode:** Immediately (responde 200 imediatamente) ou Last Node

---

## URLs do Webhook

| Modo | URL |
|------|-----|
| **Teste** (workflow ativo manualmente) | `https://n8n.host/webhook-test/SEU_PATH` |
| **Produção** (workflow publicado) | `https://n8n.host/webhook/SEU_PATH` |

> ⚠️ Use a URL de **produção** no Evolution API. A URL de teste só funciona com o workflow aberto no editor.

---

## Exemplo: Receber webhook da Evolution API

### 1. Configurar Webhook node

```
Method:    POST
Path:      whatsapp-pv360
Response:  Immediately (status 200)
Auth:      Header Auth
  Header:  apikey
  Value:   suporte123
```

URL resultante: `https://vpautomation-n8n.host/webhook/whatsapp-pv360`

### 2. Configurar no Evolution API

```bash
curl -X POST http://72.61.48.156:8080/webhook/set/pv360 \
  -H "Content-Type: application/json" \
  -H "apikey: suporte123" \
  -d '{
    "url": "https://vpautomation-n8n.host/webhook/whatsapp-pv360",
    "enabled": true,
    "webhookByEvents": false,
    "events": ["MESSAGES_UPSERT", "CONNECTION_UPDATE"]
  }'
```

### 3. Filtrar mensagens no workflow

Após o Webhook node, adicionar um **IF node**:

```
Condition: {{$json.event}} equals "messages.upsert"
AND
Condition: {{$json.data.key.fromMe}} equals false
AND  
Condition: {{$json.data.key.remoteJid}} not contains "@g.us"  (excluir grupos)
```

---

## Estrutura do payload recebido (MESSAGES_UPSERT)

Após o Webhook node, os dados ficam disponíveis em `$json`:

```javascript
// Acessar na expressão n8n:
{{ $json.event }}                          // "messages.upsert"
{{ $json.instance }}                       // "pv360"
{{ $json.data.key.remoteJid }}             // "5511999999999@s.whatsapp.net"
{{ $json.data.key.fromMe }}                // false
{{ $json.data.pushName }}                  // "João Cliente"
{{ $json.data.message.conversation }}      // "Texto da mensagem"
{{ $json.data.messageTimestamp }}          // timestamp Unix
```

---

## Workflow completo: WhatsApp → Notificação Telegram

```
[Webhook: pv360]
    ↓
[IF: é mensagem nova de cliente?]
  event == "messages.upsert"
  fromMe == false
  remoteJid não contém @g.us
    ↓ TRUE
[Set: formatar dados]
  - phone: {{ $json.data.key.remoteJid.replace('@s.whatsapp.net','') }}
  - name: {{ $json.data.pushName }}
  - text: {{ $json.data.message.conversation }}
    ↓
[Telegram: enviar notificação]
  Chat ID: -1001234567890
  Message: "📱 Nova msg de {{ $node.Set.json.name }}\n{{ $node.Set.json.text }}"
```

---

## HTTP Request node — Enviar via Evolution API

Para responder ao cliente a partir de um workflow n8n:

```
Node: HTTP Request
Method: POST
URL: http://72.61.48.156:8080/message/sendText/pv360
Headers:
  Content-Type: application/json
  apikey: suporte123
Body (JSON):
{
  "number": "{{ $json.data.key.remoteJid.replace('@s.whatsapp.net','').replace('@lid','') }}",
  "text": "Olá {{ $json.data.pushName }}! Recebemos sua mensagem e retornaremos em breve."
}
```

---

## Autenticação no Webhook

### Header Auth (recomendado)

```
Header Name:  apikey
Header Value: suporte123
```

Evolution API inclui `apikey` no payload do webhook — n8n verifica automaticamente.

### Basic Auth

```
User:     n8n
Password: senha_secreta
```

Configurar na URL do webhook no Evolution: `https://user:senha@n8n.host/webhook/path`

---

## Dicas importantes

1. **Sempre publicar o workflow** antes de configurar no Evolution API (URL de teste ≠ produção)
2. **Response mode "Immediately"** — Evolution API não aguarda resposta do webhook, mas boas práticas ditam responder 200 rápido
3. **Limitar eventos** — Configure apenas os eventos necessários no Evolution API para reduzir ruído
4. **Evitar loops** — Se o n8n envia mensagens via Evolution API, o evento `SEND_MESSAGE` pode acionar o webhook novamente. Filtrar `fromMe == true`.
