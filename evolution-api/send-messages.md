# Evolution API v2 — Envio de Mensagens

> Fonte: [doc.evolution-api.com — Message Controller](https://doc.evolution-api.com/v2/api-reference/message-controller/send-text.md)

**Base URL:** `http://SEU_HOST:8080`  
**Auth header:** `apikey: SUA_API_KEY`

---

## Enviar Texto

```
POST /message/sendText/{instance}
```

**Headers:**
```
Content-Type: application/json
apikey: suporte123
```

**Body:**
```json
{
  "number": "5511997663780",         // Número com DDI+DDD, sem + ou @
  "text": "Olá, em que posso ajudar?",
  "delay": 1000,                      // ms antes de enviar (opcional)
  "linkPreview": true,                // Preview de links (opcional)
  "quoted": {                         // Responder mensagem específica (opcional)
    "key": { "id": "BAE594145F4C59B4" },
    "message": { "conversation": "Texto da msg original" }
  }
}
```

**Resposta (201):**
```json
{
  "key": {
    "remoteJid": "5511997663780@s.whatsapp.net",
    "fromMe": true,
    "id": "BAE594145F4C59B4"
  },
  "message": {
    "extendedTextMessage": { "text": "Olá, em que posso ajudar?" }
  },
  "messageTimestamp": "1717689097",
  "status": "PENDING"
}
```

---

## Enviar Mídia (imagem, vídeo, documento)

```
POST /message/sendMedia/{instance}
```

```json
{
  "number": "5511997663780",
  "mediatype": "image",              // image | video | document | audio
  "mimetype": "image/jpeg",
  "caption": "Legenda da imagem",
  "media": "https://url-da-imagem.jpg",   // URL pública OU base64
  "fileName": "arquivo.jpg"          // Obrigatório para document
}
```

---

## Enviar Áudio (PTT — push-to-talk)

```
POST /message/sendAudio/{instance}
```

```json
{
  "number": "5511997663780",
  "audio": "https://url-do-audio.ogg",   // URL ou base64
  "encoding": true                        // Converte para formato WhatsApp
}
```

---

## Enviar Localização

```
POST /message/sendLocation/{instance}
```

```json
{
  "number": "5511997663780",
  "latitude": -23.550520,
  "longitude": -46.633308,
  "name": "Escritório VerticalParts",
  "address": "Av. Paulista, 1000, São Paulo - SP"
}
```

---

## Enviar Contato

```
POST /message/sendContact/{instance}
```

```json
{
  "number": "5511997663780",
  "contact": [
    {
      "fullName": "Gelson Simões",
      "wuid": "5511997663780",
      "phoneNumber": "+55 (11) 99766-3780"
    }
  ]
}
```

---

## Enviar Botões

```
POST /message/sendButton/{instance}
```

```json
{
  "number": "5511997663780",
  "title": "Título da mensagem",
  "description": "Corpo da mensagem",
  "footer": "Rodapé",
  "buttons": [
    { "type": "reply", "displayText": "Opção 1", "id": "opcao_1" },
    { "type": "reply", "displayText": "Opção 2", "id": "opcao_2" }
  ]
}
```

---

## Verificar se número existe no WhatsApp

```
POST /chat/whatsappNumbers/{instance}
```

```json
{
  "numbers": ["5511997663780", "5511988887777"]
}
```

**Resposta:**
```json
[
  { "exists": true, "jid": "5511997663780@s.whatsapp.net", "number": "5511997663780" },
  { "exists": false, "jid": "5511988887777@lid", "number": "5511988887777" }
]
```

> ⚠️ **Atenção:** Números que retornam `@lid` **não podem** receber mensagens via REST.
> Ver [at-lid-limitation.md](at-lid-limitation.md).

---

## Marcar como lido

```
POST /chat/markMessageAsRead/{instance}
```

```json
{
  "readMessages": [
    {
      "id": "BAE594145F4C59B4",
      "fromMe": false,
      "remoteJid": "5511997663780@s.whatsapp.net"
    }
  ]
}
```

---

## Buscar histórico de mensagens

```
POST /chat/findMessages/{instance}
```

```json
{
  "where": {
    "key": { "remoteJid": "5511997663780@s.whatsapp.net" }
  },
  "limit": 20
}
```

---

## Exemplos cURL (instância pv360 na VP)

```bash
BASE="http://72.61.48.156:8080"
KEY="suporte123"
INSTANCE="pv360"

# Enviar texto
curl -X POST "$BASE/message/sendText/$INSTANCE" \
  -H "Content-Type: application/json" \
  -H "apikey: $KEY" \
  -d '{"number":"5511999999999","text":"Olá!"}'

# Verificar se número existe
curl -X POST "$BASE/chat/whatsappNumbers/$INSTANCE" \
  -H "Content-Type: application/json" \
  -H "apikey: $KEY" \
  -d '{"numbers":["5511999999999"]}'

# Histórico de mensagens
curl -X POST "$BASE/chat/findMessages/$INSTANCE" \
  -H "Content-Type: application/json" \
  -H "apikey: $KEY" \
  -d '{"where":{"key":{"remoteJid":"5511999999999@s.whatsapp.net"}},"limit":10}'
```
