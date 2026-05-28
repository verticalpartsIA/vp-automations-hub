# n8n — Workflow Automation

> **n8n** é uma plataforma de automação de workflows open-source com interface visual drag-and-drop, suporte a 400+ integrações e capacidades de IA.

- GitHub: [n8n-io/n8n](https://github.com/n8n-io/n8n)
- Docs: [docs.n8n.io](https://docs.n8n.io/)
- Imagem Docker: `n8nio/n8n`

---

## Conceitos principais

### Workflow

Sequência automatizada de passos. Cada workflow tem:
- **Trigger** — evento que inicia a execução
- **Nodes** — ações a executar
- **Connections** — ligações entre nodes

### Nodes

Building blocks do workflow. Categorias:

| Categoria | Exemplos |
|-----------|---------|
| **Triggers** | Webhook, Schedule, Email, Telegram |
| **Core** | Code, Filter, Merge, Split, Set, HTTP Request |
| **Apps** | Gmail, Slack, HubSpot, PostgreSQL |
| **AI/LangChain** | OpenAI, Anthropic Claude, Pinecone |

### Credentials

Armazenamento seguro de autenticação. As credenciais são criptografadas e reutilizáveis entre workflows.

---

## Instância VP (produção)

```
Container: vpautomation-n8n
Porta:     5678
URL:       http://72.61.48.156:5678 (acesso interno)
Rede:      vp-automation
```

---

## Casos de uso na VP

| Caso de uso | Status |
|------------|--------|
| Receber webhook Evolution API | 🔜 Planejado |
| Notificar Slack/Telegram de novos tickets | 🔜 Planejado |
| Processar fila de atendimento WhatsApp | 🔜 Planejado |
| Sincronizar dados ERP ↔ Supabase | 🔜 Planejado |
| Relatórios agendados | 🔜 Planejado |

---

## Documentação neste diretório

| Arquivo | Conteúdo |
|---------|----------|
| [docker-setup.md](docker-setup.md) | docker-compose + variáveis de ambiente |
| [webhook-nodes.md](webhook-nodes.md) | Como usar Webhook como trigger |
| [verticalparts-setup.md](verticalparts-setup.md) | Nossa instância em produção |
