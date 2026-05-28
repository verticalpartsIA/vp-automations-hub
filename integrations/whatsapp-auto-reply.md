# Integração — Auto-Reply Claude via WhatsApp

> Como o pv360 usa Claude para responder automaticamente clientes no WhatsApp

---

## Arquitetura

```
Mensagem WhatsApp
      ↓
Evolution API webhook → server.mjs
      ↓
automateIncoming()
      ↓
callClaudeWithHistory()
      ├── Busca últimas 20 msgs do contato (Supabase)
      ├── Chama Anthropic API
      └── Envia resposta via Evolution API
```

---

## Ativar/Desativar

```bash
# No servidor Hostinger (via SSH ou painel de env vars)
HERMES_AUTO_REPLY=true    # Ativar
HERMES_AUTO_REPLY=false   # Desativar (padrão atual)

# Verificar estado atual
curl https://posvenda360.vpsistema.com/api/whatsapp/status
```

---

## Modelo Claude

```bash
# Configurar modelo (no .env do servidor)
HERMES_MODEL=claude-haiku-4-5    # Padrão — rápido e econômico
# HERMES_MODEL=claude-sonnet-4   # Mais capaz, mais caro

# Testar conexão
curl https://posvenda360.vpsistema.com/api/whatsapp/test-claude
```

---

## Implementação em server.mjs

```javascript
async function callClaudeWithHistory(remoteJid) {
  // 1. Buscar histórico do Supabase
  const { data: msgs } = await supabase
    .from("whatsapp_messages")
    .select("body, from_me, push_name, created_at")
    .eq("remote_jid", remoteJid)
    .order("created_at", { ascending: false })
    .limit(20);

  if (!msgs?.length) return;

  // 2. Reverter para ordem cronológica
  msgs.reverse();

  // 3. Formatar mensagens para Claude
  const messages = msgs.map(m => ({
    role: m.from_me ? "assistant" : "user",
    content: m.body || ""
  }));

  // 4. Chamar Anthropic API
  const response = await anthropic.messages.create({
    model: process.env.HERMES_MODEL || "claude-haiku-4-5",
    max_tokens: 500,
    system: "Você é um assistente de atendimento ao cliente da VerticalParts...",
    messages
  });

  const replyText = response.content[0].text;

  // 5. Enviar via Evolution API
  await fetch(`http://72.61.48.156:8080/message/sendText/pv360`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "apikey": process.env.EVOLUTION_APIKEY || "suporte123"
    },
    body: JSON.stringify({ number: remoteJid, text: replyText })
  });

  // 6. Salvar resposta no Supabase
  await supabase.from("whatsapp_messages").insert({
    instance: "pv360",
    remote_jid: remoteJid,
    from_me: true,
    body: replyText,
    // ...
  });
}
```

---

## Limitações conhecidas

| Limitação | Detalhes |
|-----------|---------|
| **@lid não funciona** | Evolution API rejeita envio para @lid. Ver [at-lid-limitation.md](../evolution-api/at-lid-limitation.md) |
| **Histórico limitado** | Apenas últimas 20 mensagens no contexto |
| **Sem memória entre instâncias** | Cada instância de server.mjs não compartilha estado |
| **Fire-and-forget** | Erros de envio são logados mas não propagados ao cliente |

---

## Sistema de prompts sugerido

```javascript
const systemPrompt = `
Você é um assistente de atendimento ao cliente da VerticalParts, 
distribuidora de autopeças localizada em São Paulo.

Diretrizes:
- Responda sempre em português, de forma cordial e objetiva
- Para dúvidas sobre peças específicas: solicite marca/modelo/ano do veículo
- Para status de pedidos: informe que irá verificar e retornar
- Não faça promessas de preço ou disponibilidade sem confirmar
- Se não souber responder: "Vou verificar com nossa equipe e retorno em breve"
- Mantenha respostas concisas (máximo 3 parágrafos)

Horário de atendimento: Segunda a Sábado, 8h às 18h
Telefone direto: (11) 99766-3780
`;
```

---

## Notificação de nova mensagem (NOTIFY_WEBHOOK_URL)

```bash
# .env
NOTIFY_WEBHOOK_URL=https://hooks.slack.com/services/T.../B.../...
# ou
NOTIFY_WEBHOOK_URL=https://vpautomation-n8n.host/webhook/whatsapp-notify
```

Quando configurado, o server.mjs envia POST com:
```json
{
  "event": "new_message",
  "phone": "5511999999999",
  "name": "João Cliente",
  "text": "Olá, preciso de uma peça",
  "ticket_id": "uuid-do-ticket"
}
```

---

## Monitoramento

```bash
# Ver últimas respostas automáticas no Supabase
SELECT body, created_at
FROM whatsapp_messages
WHERE from_me = true
  AND raw->>'author' = 'Claude (VerticalParts Bot)'
ORDER BY created_at DESC
LIMIT 10;

# Verificar status do servidor
curl https://posvenda360.vpsistema.com/api/whatsapp/status

# Testar Claude
curl https://posvenda360.vpsistema.com/api/whatsapp/test-claude
```
