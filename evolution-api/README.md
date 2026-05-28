# Evolution API v2

> **Evolution API** é uma plataforma de integração de mensagens open-source, focada em WhatsApp via protocolo Baileys, com suporte a Typebot, Chatwoot, n8n, OpenAI, Dify e outros.

- GitHub: [EvolutionAPI/evolution-api](https://github.com/EvolutionAPI/evolution-api)
- Docs: [doc.evolution-api.com](https://doc.evolution-api.com/)
- Imagem Docker: `atendai/evolution-api:v2.x.x`

---

## Arquitetura

```
Cliente WhatsApp
      ↓ mensagem
Evolution API (container Docker)
  ├── Instância (ex: pv360)
  │   ├── Conexão WhatsApp via Baileys
  │   ├── Webhook → seu endpoint
  │   └── REST API → enviar mensagens
  └── PostgreSQL (storage) + Redis (cache)
```

---

## Conceitos principais

### Instância

Uma **instância** = uma conexão WhatsApp. Cada instância representa um número de WhatsApp conectado.

```
instância "pv360" ← número (11) 99766-3780 conectado
```

### JID (Jabber ID)

Identificador único de cada conversa no WhatsApp:

| Formato | Tipo | Exemplo |
|---------|------|---------|
| `5511999999999@s.whatsapp.net` | Contato normal | Suportado ✅ |
| `178288890790057@lid` | Multi-device privacy | Limitado ⚠️ |
| `5511999999999-1600000000@g.us` | Grupo | Suportado ✅ |

> Ver [at-lid-limitation.md](at-lid-limitation.md) para detalhes sobre o problema `@lid`.

### API Key

Autenticação via header `apikey: SUA_CHAVE`. Configurada em `AUTHENTICATION_API_KEY` no `.env`.

---

## Versões em produção na VP

| Container | Porta | API Key | Instância | Status |
|-----------|-------|---------|-----------|--------|
| `evolution-api` | 8080 | `suporte123` | `pv360` | ✅ Ativo — pv360 conectado |
| `evolution_api` | 8081 | `VPEvol2026key` | — | 📦 Reserva — vazio |

> ⚠️ **NÃO mover pv360 para 8081** sem planejamento — requer reconexão WhatsApp com QR Code.

---

## Documentação neste diretório

| Arquivo | Conteúdo |
|---------|----------|
| [docker-setup.md](docker-setup.md) | docker-compose + variáveis de ambiente |
| [instance-management.md](instance-management.md) | Criar, conectar, gerenciar instâncias |
| [send-messages.md](send-messages.md) | Endpoints de envio |
| [webhooks.md](webhooks.md) | Eventos, configuração, payloads |
| [at-lid-limitation.md](at-lid-limitation.md) | Problema @lid — limitação WhatsApp |
| [verticalparts-pv360.md](verticalparts-pv360.md) | Nossa instância em produção |
