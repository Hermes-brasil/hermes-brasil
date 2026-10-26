# Instalação básica do Hermes Agent

Guia para colocar o Hermes Agent rodando em **Linux, macOS ou Windows (WSL2)** em poucos minutos, com um provedor de modelo configurado e a primeira conversa funcionando.

> Para servidor de produção (systemd, swap, backup, hardening) use o [guia de produção em VPS](./instalacao-producao-vps.md) depois deste.

## Pré-requisitos

- Terminal com `curl` e `git`
- ~2 GB livres em disco
- Linux: Ubuntu 22.04+ / Debian 12+ (outras distros funcionam, o instalador resolve as dependências)
- macOS: versões recentes, Intel ou Apple Silicon
- Windows: **use WSL2** (veja a seção Windows abaixo)

Não é necessário instalar Python antes. O instalador cuida de `uv`, do Python, da venv e do launcher `hermes`.

## 1. Instalar

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

Ao terminar, feche e reabra o terminal (ou rode `source ~/.bashrc`) para o comando `hermes` entrar no PATH.

## 2. Conferir a instalação

```bash
hermes --version   # versão e canal (stable/dev)
hermes doctor      # checa configuração, dependências e o que estiver faltando
```

`hermes doctor` é o primeiro lugar a olhar quando algo não funciona: ele aponta o que está faltando em vez de você adivinhar.

## 3. Configurar provedor e modelo

```bash
hermes setup   # wizard: escolhe provedor, pede a chave e define o modelo
```

Depois, para trocar só o modelo:

```bash
hermes model
```

Onde as coisas ficam:

- **Segredos (chaves de API)** → `~/.hermes/.env`
- **Configuração** → `~/.hermes/config.yaml`

Duas regras que evitam a maioria dos problemas:

1. Chave de API vai no `.env`, nunca no `config.yaml`.
2. Não edite o `config.yaml` na mão para mudar uma opção — use `hermes config set CHAVE VALOR`, que valida antes de gravar. Uma indentação errada no YAML derruba o gateway em execução.

## 4. Primeira conversa

```bash
hermes                    # chat interativo
hermes chat -q "resuma o README deste repositório"
hermes --continue         # retoma a última sessão
```

## 5. Conectar mensageria (opcional)

Para falar com o agente pelo Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Teams e outros:

```bash
hermes gateway setup     # configura as plataformas
hermes gateway install   # instala como serviço (systemd no Linux, launchd no macOS)
hermes gateway status    # confere se subiu
```

Em **WSL e Docker** não use `install`: rode em foreground com `hermes gateway run`.

Logs ficam em `~/.hermes/logs/`.

## Windows

- **WSL2 (recomendado):** instale o Hermes **dentro do WSL**, não no PowerShell. É o caminho com menos atrito.
- **Nativo:** funciona, mas tem ajustes próprios de console/UTF-8 e caminhos.

Qualquer um dos dois modos tem pegadinhas específicas — vale ler a doc oficial do modo escolhido antes de começar.

## Onde ficam seus dados

```
~/.hermes/config.yaml       configuração (sem segredos)
~/.hermes/.env              chaves de API (só segredos)
~/.hermes/skills/           skills instaladas
~/.hermes/state.db          sessões (SQLite com busca)
~/.hermes/sessions/         transcrições e índices do gateway
~/.hermes/logs/             logs do gateway e de erros
~/.hermes/profiles/<nome>/  instâncias isoladas (config, skills e memória próprias)
```

Se você usa **profiles**, o diretório real é o de `$HERMES_HOME`, não `~/.hermes`. Nunca hardcode o caminho em scripts.

## Problemas comuns

- **`hermes: command not found`** — o PATH não foi recarregado. Reabra o terminal ou rode `source ~/.bashrc`. Em instalações por usuário, o launcher fica em `~/.local/bin`.
- **Erro de autenticação/modelo** — rode `hermes doctor` e confira se a chave está no `~/.hermes/.env` (não no `config.yaml`).
- **Gateway não responde** — `hermes gateway status`, depois `hermes gateway restart`. Se persistir, a causa está em `~/.hermes/logs/`.
- **Aviso do "Claude Subscription DirectSDK"** — aparece quando o Claude Code não está instalado. É inofensivo se você não usa a assinatura Claude; pode ignorar.
- **Atualizar depois** — a doc de *Updating* cobre atualização e desinstalação; a mesma URL do instalador reinstala/atualiza.

## Próximos passos

- [Produção em VPS](./instalacao-producao-vps.md) — systemd, swap, backup, acesso remoto
- [Seu primeiro agente](./primeiro-agente.md) — do chat vazio a um agente útil
- [MCP no Hermes](./mcp-no-hermes.md) — plugar ferramentas externas
- [Profiles e Kanban](./profiles-e-kanban.md) — várias instâncias e trabalho paralelo
- [Autoria de skills](./autoria-de-skills.md) — por que a sua skill não carrega

## Documentação oficial

- Docs: <https://hermes-agent.nousresearch.com/docs/>
- Instalação: <https://hermes-agent.nousresearch.com/docs/getting-started/installation>
- Repositório: <https://github.com/NousResearch/hermes-agent>

Dúvidas: abra uma issue aqui ou use as [Discussions da comunidade](https://github.com/Hermes-brasil/hermes-brasil/discussions).
