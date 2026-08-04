# Guia básico: instalando o Hermes Agent

Para um caminho **de produção em VPS** (gateway 24/7, update, health-check), use:

→ **[instalacao-producao-vps.md](./instalacao-producao-vps.md)**

Este arquivo é o atalho mínimo.

## Instalação

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
hash -r
hermes version
hermes doctor
```

## Primeiros comandos

```bash
hermes setup          # wizard
hermes model          # escolhe modelo/provedor
hermes                # chat interativo
hermes chat -q "ping" # one-shot
```

## Gateway (Telegram etc.)

```bash
hermes gateway setup
hermes gateway install
hermes gateway start
hermes gateway status
```

## Recursos

- Produção VPS: [instalacao-producao-vps.md](./instalacao-producao-vps.md)
- Cron: [cron-em-producao.md](./cron-em-producao.md)
- Profiles + Kanban: [profiles-e-kanban.md](./profiles-e-kanban.md)
- Docs oficiais: https://hermes-agent.nousresearch.com/docs/
- Repo: https://github.com/NousResearch/hermes-agent

Dúvidas: issues neste repositório ou canais da comunidade Hermes Brasil (README).
