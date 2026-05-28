# Evolution API — Limitação @lid (WhatsApp Multi-Device Privacy)

> Problema identificado e documentado em: 2026-05-28
> Impacto: **86%** das conversas pv360 afetadas (ambiente de desenvolvimento)

---

## O que é @lid

Quando um contato WhatsApp usa **dispositivos vinculados** (tablet, web, segundo celular), suas mensagens chegam com um identificador `@lid` (Linked ID) em vez do número de telefone padrão.

| JID Normal | JID @lid |
|-----------|---------|
| `5511997663780@s.whatsapp.net` | `178288890790057@lid` |
| Telefone visível | **Número completamente oculto** |

---

## Por que acontece no ambiente de desenvolvimento pv360

O número conectado ao pv360 é o **pessoal do Gelson** (5511997663780). Todos os seus contatos pessoais que usam multi-device aparecem como `@lid`.

**Em produção com número dedicado:** apenas clientes que explicitamente usam multi-device virariam `@lid` — seria uma minoria.

### Estatísticas atuais (ambiente dev)

| JID | Conversas | Mensagens |
|-----|-----------|-----------|
| `@lid` | 26 (86%) | 503 |
| `@s.whatsapp.net` | 4 (14%) | 10 |

---

## O que FUNCIONA para @lid

| Operação | Status |
|----------|--------|
| Receber mensagens via webhook | ✅ Funciona |
| Salvar no Supabase | ✅ Funciona |
| Criar tickets automaticamente | ✅ Funciona |
| Listar no painel `/whatsapp-threads` | ✅ Funciona |
| Mostrar histórico no painel | ✅ Funciona |

---

## O que NÃO FUNCIONA para @lid

| Operação | Status |
|----------|--------|
| Enviar resposta via Evolution API REST | ❌ Falha com 400 |
| Auto-reply Claude | ❌ Silenciosamente não enviado |
| Notificar cliente | ❌ Impossível |

---

## Por que a Evolution API rejeita @lid

Antes de enviar, a Evolution API faz uma verificação de existência:

```
POST /message/sendText/pv360
{ "number": "178288890790057@lid", "text": "Olá" }
```

**Resposta 400:**
```json
{
  "status": 400,
  "error": "Bad Request",
  "response": {
    "message": [
      {
        "exists": false,
        "jid": "178288890790057@lid"
      }
    ]
  }
}
```

A mensagem **não é enviada**. Não há fallback. O número de telefone real associado ao `@lid` não está disponível no payload do webhook — o WhatsApp esconde completamente.

---

## Fix aplicado no pv360 (2026-05-28)

**Arquivo:** `nodejs/src/routes/_app/thread.$id.tsx`

**Antes:** Área de input habilitada para @lid. O `server.mjs` salvava a mensagem localmente (`lid_local_only: true` no campo `raw`) e retornava `{ok: true}` — a UI mostrava como "enviado" mas o cliente nunca recebia.

**Depois:**
- Para `@lid`: área de input substituída por painel informativo âmbar
- Para contatos normais: comportamento inalterado

```jsx
{remoteJid.endsWith("@lid") ? (
  <div className="rounded-lg bg-amber-500/8 border border-amber-500/20 px-4 py-3 text-center">
    <p className="text-xs font-medium text-amber-600">
      Contato via dispositivo vinculado (@lid)
    </p>
    <p className="mt-1 text-[11px] text-muted-foreground">
      O WhatsApp oculta o número deste contato — respostas por aqui não são entregues.
      Para responder, use o WhatsApp diretamente no número (11) 99766-3780.
    </p>
  </div>
) : (
  // Input normal + botão enviar
)}
```

**Commit:** `7b0de32` | **Branch:** `main` | **Repo:** `verticalpartsIA/resolve-360`

---

## Solução definitiva: número dedicado

| Prazo | Ação |
|-------|------|
| ✅ Curto prazo | UI informa limitação, orienta usar WhatsApp diretamente |
| 🔜 Médio prazo | Conectar número DEDICADO ao pv360 |

### Por que o número dedicado resolve

Com um **chip exclusivo de atendimento** (não o pessoal do Gelson), os clientes não estão na agenda pessoal de ninguém. Isso faz com que seus JIDs apareçam como `@s.whatsapp.net` mesmo se usarem multi-device — o `@lid` só aparece para contatos da agenda pessoal do número conectado.

### Processo de reconexão com novo número

```bash
# 1. Desconectar instância atual (opcional)
curl -X DELETE http://72.61.48.156:8080/instance/logout/pv360 \
  -H "apikey: suporte123"

# 2. Conectar com novo número (QR Code)
# Abrir no navegador:
http://72.61.48.156:8080/instance/connect/pv360
# → Escanear QR com o NOVO número

# 3. Verificar conexão
curl http://72.61.48.156:8080/instance/connectionState/pv360 \
  -H "apikey: suporte123"
```

---

## Identificando @lid no código

```javascript
// Verificar se é @lid
const isLid = remoteJid.endsWith("@lid");

// No webhook, @lid aparece exatamente assim:
// data.key.remoteJid = "178288890790057@lid"

// Não há campo com o número real — o WhatsApp não expõe
```

---

## Campo raw.lid_local_only

O `server.mjs` salva mensagens de operadores para @lid com flag especial:

```json
{
  "raw": {
    "lid_local_only": true,
    "note": "WhatsApp @lid - mensagem salva localmente, NÃO enviada"
  }
}
```

Isso permite identificar no banco quais mensagens nunca foram entregues.
