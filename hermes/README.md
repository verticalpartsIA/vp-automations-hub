# Hermes AI Agent

> **Hermes** é um agente autônomo desenvolvido pela [Nous Research](https://nousresearch.com) que cresce com o uso — cria skills a partir da experiência, melhora durante a execução e mantém memória entre sessões.

- GitHub: [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
- Docs: [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/)
- Imagem Docker: `ghcr.io/hostinger/hvps-hermes-agent:latest`

---

## O que é

Diferente de copilots vinculados a IDEs ou chatbots simples, o Hermes **opera de forma independente** em diversas infraestruturas (VPS local, Docker, SSH, serverless). Ele suporta 20+ plataformas de mensagens e possui um sistema de skills criadas pelo próprio agente.

### Características principais

| Feature | Detalhe |
|---------|---------|
| **Gateway de mensagens** | Telegram, Discord, Slack, WhatsApp, Teams, Signal, CLI |
| **Memória persistente** | FTS5 full-text search + summarização LLM |
| **Terminal backends** | local, Docker, SSH, Daytona, Singularity, Modal |
| **Skills** | Criadas autonomamente, reutilizadas entre sessões |
| **Subagentes** | Spawn de agentes paralelos para workstreams |
| **Cron** | Automações agendadas (daily standups, health checks) |
| **60+ ferramentas** | Web search, visão, geração de imagem, TTS |

---

## Arquitetura interna (nosso container)

```
Container: vpautomation-hermes
Imagem:    ghcr.io/hostinger/hvps-hermes-agent:latest
URL:       https://vpautomation-hermes.srv1510643.hstgr.cloud

Processos internos:
  PID 1:  ttyd (web terminal — porta 4860) → interface de controle
  PID 10: /opt/hermes/.venv/bin/python3 ... hermes gateway run

Volume montado:
  /opt/data  ←→  /docker/vpautomation-hermes/data/ (host)
```

---

## Instalação (Linux/macOS/WSL2)

```bash
# Instala em ~60 segundos
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash

# Setup OAuth (Nous Portal — modelos + ferramentas)
hermes setup --portal

# Ou configuração manual
cp ~/.hermes/.env.example ~/.hermes/.env
# Editar .env com as chaves
```

---

## Estrutura de arquivos

```
~/.hermes/   (ou /opt/data/ dentro do container)
├── config.yaml    ← Configuração principal (NÃO contém secrets)
├── .env           ← API keys e tokens (NUNCA commitar)
├── auth.json      ← OAuth credentials
├── SOUL.md        ← Identidade/personalidade do agente
├── memories/
│   └── main.md    ← Memória acumulada (limpar se > 2200 chars)
├── skills/        ← Skills criadas pelo agente
├── cron/          ← Jobs agendados
├── sessions/      ← Sessões de gateway (Telegram, etc.)
└── logs/
    └── gateway.log ← Log principal do gateway
```

---

## Documentação neste diretório

| Arquivo | Conteúdo |
|---------|----------|
| [config-reference.md](config-reference.md) | Referência completa do `config.yaml` |
| [telegram-setup.md](telegram-setup.md) | Setup do gateway Telegram |
| [docker-deployment.md](docker-deployment.md) | Backend Docker para terminal |
| [soul-template.md](soul-template.md) | Template SOUL.md para VP |
| [verticalparts-setup.md](verticalparts-setup.md) | Nossa instância em produção |
