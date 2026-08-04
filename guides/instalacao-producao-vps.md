# Instalação do Hermes Agent em produção (VPS)

Guia prático para rodar o Hermes Agent 24/7 em VPS Ubuntu (ou similar). Foco em instalação real, gateway de mensagens e health-check — não em demo local.

Docs oficiais (sempre preferir quando divergir): [hermes-agent.nousresearch.com](https://hermes-agent.nousresearch.com/docs/)

## Pré-requisitos

- Ubuntu 22.04+ (ou Debian recente)
- Acesso root ou sudo
- ~2 GB RAM mínimo (4 GB+ confortável com vários profiles)
- Python 3.11 (o installer gerencia via uv)
- Conta em pelo menos um provedor de modelo (OpenRouter, DeepSeek, Anthropic, xAI, etc.)

## 1. Instalar

```bash
# Método oficial (recomendado)
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash

# Depois recarregue o PATH se o shell não achar o comando
hash -r
hermes version
hermes doctor
```

O que o installer faz (resumo):

- Clona o código (em root costuma ir para `/usr/local/lib/hermes-agent`)
- Cria venv e instala dependências
- Deixa o comando `hermes` no PATH
- Dados e config ficam em `~/.hermes/` (`HERMES_HOME`)

**Não use** `pipx install hermes-agent` nem `pip install` do sistema como caminho principal — o fluxo suportado é o `install.sh` (ou clone + venv do projeto).

## 2. Setup inicial

```bash
hermes setup          # wizard interativo (modelo, terminal, gateway, tools)
# ou por seção:
hermes setup model
hermes setup gateway
hermes model          # picker de modelo/provedor
```

Chaves de API vão em `~/.hermes/.env` (nunca no git, nunca em skill pública).

Config estrutural fica em `~/.hermes/config.yaml`:

```bash
hermes config path
hermes config set model.default "nome-do-modelo"
hermes config check
```

## 3. Gateway (Telegram e outros)

```bash
hermes gateway setup    # conecta plataformas
hermes gateway install  # serviço em background (quando systemd user bus existe)
hermes gateway start
hermes gateway status
```

Plataformas comuns: Telegram, Discord, Slack, WhatsApp, Signal, API server.

### Headless / sem TUI

Se não houver terminal interativo, edite `~/.hermes/.env` com as variáveis da plataforma e suba o gateway:

```bash
hermes gateway install
hermes gateway start
# ou foreground para debug:
hermes gateway run
```

Variáveis por plataforma: docs de messaging + skill oficial `hermes-agent` → `references/gateway-noninteractive-setup.md`.

### Telegram — checklist rápido

1. Crie o bot com `@BotFather`, copie o token
2. Coloque `TELEGRAM_BOT_TOKEN=...` no `.env`
3. Inicie uma conversa com o bot (ou adicione a um grupo)
4. Defina home channel se quiser destino padrão de crons: no chat, `/sethome` (gateway)
5. Teste:

```bash
hermes send --to telegram:<chat_id> "ping"
# tópico de fórum:
hermes send --to telegram:<chat_id>:<thread_id> "ping thread"
```

## 4. Health-check mínimo (faça isso sempre)

```bash
hermes version
hermes doctor --fix
hermes status --all
tail -50 ~/.hermes/logs/gateway.log
tail -30 ~/.hermes/logs/errors.log
```

Sinais de vida bons:

- `hermes version` responde com versão + commit
- Gateway com processo `gateway run` ativo
- `hermes send` entrega mensagem no app

## 5. Update em produção

```bash
hermes update
hermes version          # confira se subiu de verdade
hermes doctor --fix
hermes gateway restart  # se o gateway não relançou sozinho
```

O update costuma:

1. Backup em `~/.hermes/backups/pre-update-...`
2. `git pull` no source
3. Reinstall no venv
4. Rebuild da web UI (se aplicável)

**Pitfall:** backup grande pode estourar timeout do comando enquanto pull/install já terminaram. Sempre valide com `hermes version`.

## 6. Systemd user bus ausente (container / alguns VPS)

Se aparecer:

```text
Failed to connect to bus: No medium found
```

`systemctl --user` não é o caminho. Opções:

1. Habilitar linger + systemd user (quando a imagem permite): `loginctl enable-linger $USER`
2. Rodar gateway via supervisor do próprio Hermes / processo gerenciado
3. Fallback manual:

```bash
ps aux | grep 'gateway run' | grep -v grep
kill -TERM <PID>          # graceful
# relançar:
hermes gateway run        # ou o mecanismo do seu supervisor
```

## 7. Segurança básica no VPS

- SSH por chave, sem root login por senha
- Firewall (UFW): liberar só 22 (ou seu SSH) + o que o gateway/API realmente precisa
- `hermes config set security.redact_secrets true` se quiser mascarar segredos no contexto
- API server exposto na internet + terminal local = risco; só faça com auth forte e consciência
- Nunca commitar `.env`, tokens OAuth ou dumps de sessão

## 8. Próximos passos úteis

| Objetivo | Onde |
|---|---|
| Cron confiável | [cron-em-producao.md](./cron-em-producao.md) |
| Multi-agente / Kanban | [profiles-e-kanban.md](./profiles-e-kanban.md) |
| Ops dia a dia | skill `hermes-ops-producao` |
| Docs oficiais | https://hermes-agent.nousresearch.com/docs/ |

## Troubleshooting rápido

| Sintoma | Ação |
|---|---|
| `hermes: command not found` | Reabra o shell; confira PATH / symlink em `/usr/local/bin` |
| Gateway “sobe e morre” no logout SSH | `loginctl enable-linger $USER` ou supervisor de verdade |
| Telegram silencioso | Token, intents/permissões, pairing/approve se necessário |
| Modelo 401 | Chave no `.env` do profile certo; `providers:` não pode ficar `{}` sobrescrevendo o global |
| Update “não pegou” | `hermes version` + logs; não misturar pip do sistema |

---

Contribuição da comunidade Hermes Brasil com base em operação real em VPS. Melhorias via PR são bem-vindas.
