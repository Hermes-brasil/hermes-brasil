---
name: hermes-ops-producao
description: Operar Hermes Agent em produção no VPS — update, restart de gateway, health-check, profiles múltiplos e ambientes sem systemd user bus.
---

# Hermes Ops — Produção

Procedimentos para manter o Hermes Agent saudável 24/7. Use antes/depois de update, quando o gateway “some”, quando Telegram para de responder, ou quando `systemctl --user` falha.

Guia longo: `guides/instalacao-producao-vps.md` neste repositório.

## Quando carregar

- `hermes update` / dúvida se a versão subiu
- `hermes gateway restart` trava ou não relança processos
- `hermes status` ok mas mensagem não chega
- Erro `Failed to connect to bus: No medium found`
- Pós-reboot do VPS

## Update

```bash
hermes update
hermes version          # obrigatório validar
hermes doctor --fix
```

O update tipicamente:

1. Backup em `~/.hermes/backups/pre-update-YYYY-MM-DD-HHMMSS...`
2. `git pull` no source (ex.: `/usr/local/lib/hermes-agent` em install root FHS)
3. Reinstall no venv do projeto
4. Build da web UI se existir

**Pitfalls**

- Timeout do comando ≠ update falhou. Backup grande demora; confira `hermes version`.
- Nunca use `pip` do sistema. Fallback manual: `cd` no source e `./venv/bin/pip install -e .`
- Depois do update, reinicie gateway se as conexões de messaging ficarem stale

## Restart do gateway

```bash
hermes gateway restart
# se a CLI aceitar na sua versão:
hermes gateway status
ps aux | grep 'gateway run' | grep -v grep
```

Graceful shutdown pode estourar timeout da CLI enquanto o processo ainda está drenando sessões. Olhe o PID e os logs.

### Sem systemd user bus

```bash
systemctl --user stop hermes-gateway
# → Failed to connect to bus: No medium found
```

Nesse ambiente:

1. Não dependa de `systemctl --user`
2. `ps aux | grep 'gateway run'`
3. `kill -TERM <pid>` no gateway principal
4. Relançar via supervisor do Hermes, `hermes gateway run`, ou o unit/supervisor que você configurou no host

Se a imagem permitir systemd de verdade:

```bash
sudo loginctl enable-linger "$USER"
# units em ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user restart hermes-gateway
```

### SIGKILL no stop (systemd)

Se o journal mostra `status=9/KILL` no restart, o gateway passou de `TimeoutStopSec` drenando agentes. Mitigações comuns:

- Aumentar `TimeoutStopSec` no unit
- `KillMode=process` (não mata cgroup inteiro de uma vez)
- Ajustar timeouts de drain/inactivity na `config.yaml` do Hermes

## Health-check ponta a ponta

Ordem curta:

```bash
hermes version
hermes doctor --fix
hermes status --all
tail -40 ~/.hermes/logs/gateway.log
tail -20 ~/.hermes/logs/errors.log
hermes send --to telegram:CHAT_ID "ping pós-ops"
# fórum/tópico:
hermes send --to telegram:CHAT_ID:THREAD_ID "ping thread"
```

Checklist mental:

| Check | OK se |
|---|---|
| Versão | string de versão + commit recente |
| Doctor | sem erro bloqueante |
| Processo | `gateway run` presente |
| Send | mensagem chega no app |
| Cron | `hermes cron list` sem rain de `last_status` ruim |

## Avisos comuns (muitas vezes não-críticos)

- `Home-channel startup notification failed: send_path_degraded` — confirme com `hermes send`
- `API server network-accessible AND terminal backend local` — risco de desenho; não “ignore para sempre” se a API está pública
- `Secret redaction: DISABLED` — ligue se quiser: `hermes config set security.redact_secrets true`
- `Running as ROOT` — comum em VPS single-tenant; ciente do blast radius

## Profiles múltiplos

```bash
hermes profile list
ps aux | grep 'hermes.*-p \|gateway run' | grep -v grep
```

Após update/restart, confira se o **default** voltou. Workers Kanban usam `-p <profile>` no spawn — profile inexistente = task pronta que nunca anda (assignee fantasma).

Ver: `guides/profiles-e-kanban.md`

## Cron (ponte)

Jobs de produção: prompt autocontido, delivery explícito, ledger se muta o mundo, `script` = **filename** em `~/.hermes/scripts/`.

Ver: `guides/cron-em-producao.md`

## Segurança operacional rápida

- `.env` e tokens fora de git e de skill pública
- Preferir approve mode consciente em gateway (`approvals.mode`)
- Não expor API server + shell local na internet sem auth e rede restrita
- Backup de `~/.hermes` antes de migração grande (`hermes backup` se disponível na versão)

## Comandos de bolso

```bash
hermes version
hermes doctor --fix
hermes status --all
hermes update
hermes gateway status
hermes gateway restart
hermes cron list
hermes profile list
hermes logs                 # se disponível
tail -f ~/.hermes/logs/gateway.log
```

## Referências

- Docs: https://hermes-agent.nousresearch.com/docs/
- Skill oficial de produto (no seu Hermes instalado): `hermes-agent`
- Guias neste repo: `guides/instalacao-producao-vps.md`, `cron-em-producao.md`, `profiles-e-kanban.md`
