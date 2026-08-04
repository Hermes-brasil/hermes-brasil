# Cron em produção no Hermes Agent

Jobs agendados são onde o Hermes deixa de ser chat e vira planta. Este guia cobre o que quebra na prática — e como evitar.

CLI de referência:

```bash
hermes cron list
hermes cron create '0 9 * * *' --help   # veja flags da sua versão
hermes cron run <id>
hermes cron pause <id>
hermes cron remove <id>
```

Dentro de uma sessão de agente, a tool `cronjob` espelha essas operações.

## 1. Anatomia de um job sério

Todo job de produção precisa de:

1. **Schedule** claro (`30m`, `every 2h`, ou cron `0 12 * * 1-5`)
2. **Prompt autocontido** — o job roda em sessão fresca, sem memória do chat
3. **Destino de entrega** (`deliver`) — senão a saída some ou vai pro lugar errado
4. **Modo de falha legível** — log + notificação, não silêncio eterno
5. **Idempotência** — rodar 2x não deve duplicar efeito no mundo real

## 2. Prompt autocontido

Ruim:

> “Continua o que a gente falou e posta o resumo.”

Bom:

> “Você é o operador do projeto X. Leia o arquivo Y, rode o script Z, se status=ok envie resumo em bullets para o Telegram home. Se falhar, reporte stderr exato.”

Inclua no prompt:

- Caminhos absolutos ou convenção estável (`~/.hermes/...`)
- Skills a carregar (quando a tool/CLI permitir `skills`)
- Formato de saída
- O que NÃO fazer (gastar dinheiro, publicar sem gate, etc.)

## 3. Delivery

Valores úteis:

| Valor | Efeito |
|---|---|
| `origin` | Volta pro chat/topic onde o job foi criado |
| `local` | Só salva em `~/.hermes/cron/output/` (sem push) |
| `telegram:CHAT_ID` | Chat específico |
| `telegram:CHAT_ID:THREAD_ID` | Tópico de fórum Telegram |
| `all` | Fan-out para home channels conectados |

Defina delivery na criação. Jobs “sumiram” quase sempre são delivery errado ou `local` sem você olhar a pasta.

## 4. Script vs agente

Dois modos:

### A) Agente (default)

LLM roda o prompt a cada tick. Bom para sumarizar, decidir, redigir.

### B) `script` + opcionalmente `no_agent`

Roda um arquivo sob `~/.hermes/scripts/`. Bom para watchdog, métrica fixa, health-check barato.

**Regra crítica:** o campo `script` é **só o nome do arquivo** (ex.: `health_ping.sh`), nunca o corpo do bash inline.

```bash
# certo
cat > ~/.hermes/scripts/health_ping.sh << 'EOF'
#!/usr/bin/env bash
set -euo pipefail
# ... lógica ...
EOF
chmod +x ~/.hermes/scripts/health_ping.sh
# job aponta script: "health_ping.sh"
```

```text
# errado — Hermes tenta abrir um path literal com #!/bin/bash
script: "#!/bin/bash\necho hi"
```

Com `no_agent=true`:

- stdout não-vazio = mensagem entregue
- stdout vazio = silêncio (padrão watchdog)
- exit ≠ 0 = alerta de erro

## 5. HOME e credenciais de terceiros

Crons herdam o ambiente do scheduler. Se a ferramenta externa (ex.: CLI de rede social, gcloud) espera `HOME` de um usuário específico e o job roda como root com outro home, você leva `NoAuthMethod` / token sumido.

Padrão seguro:

```bash
# dentro do script
export HOME=/home/usuario_dono_do_token
# ou use caminhos absolutos para configs/creds
```

Nunca assuma que o HOME do cron = HOME do seu login SSH interativo.

## 6. Ledger antes de mutar o mundo

Se o job **publica, envia e-mail, gasta API paga ou altera produção**:

1. Consulte um ledger (arquivo/SQLite) **antes**
2. Grave intenção/resultado **depois**
3. Marque item como feito com ID externo (tweet id, URN, deploy sha)

Sem ledger, o clássico é republicar o mesmo conteúdo 3 dias seguidos com IDs diferentes.

Gate humano útil para conteúdo:

```text
status: draft → aprovado → publicado
```

Script só age em `aprovado` e grava `publicado` + id.

## 7. Teto de custo e fallback

Fallback de modelo sem teto é fogo baixo: barato cai → pago sobe → loop.

Na config / disciplina operacional:

- Limite de retries
- Limite de $ ou tokens por task/dia
- Quando estoura: **para e avisa**, não tenta “mais uma vez no silent”

Evite `openrouter/auto` (ou equivalente “surpresa cara”) como único fallback de jobs noturnos.

## 8. Toolsets enxutos

Job de health-check não precisa de browser + image gen + delegation.

Restrinja toolsets quando a CLI/tool permitir. Menos tool = menos alucinação de ferramenta e menos token.

## 9. Quando a tool `cronjob` encrenca

Sintomas: “schedule is required” em loop, estouro de iterações, prompt longo rejeitado.

Escapatória:

1. Crie o job **mínimo** (schedule + name)
2. `update` campo a campo
3. Preferir `script` para payloads longos
4. Em último caso, editar `~/.hermes/cron/jobs.json` com backup prévio — a chave primária é `id` (às vezes aparece como `job_id` na listagem)

Depois de qualquer edição manual:

```bash
hermes cron list
hermes cron run <id>    # smoke test
```

## 10. Triagem quando “não rodou” / “só erro”

Ordem:

1. `hermes cron list` — enabled? `last_status`?
2. `~/.hermes/cron/output/<id>/` — último markdown/stderr
3. `~/.hermes/logs/errors.log` e trecho recente de `gateway.log`
4. Skills inexistentes no job (`skills: ["web"]` fantasma gera ruído)
5. Corrija → `cron run` até `ok`

## 11. Checklist de job novo

- [ ] Prompt autocontido
- [ ] Schedule no fuso que você acha que está (BRT vs UTC)
- [ ] Delivery testado com `run` manual
- [ ] Ledger se muta mundo externo
- [ ] Teto de custo / sem fallback infinito
- [ ] Script = filename se usar script
- [ ] HOME/creds corretos
- [ ] Skills listadas existem de verdade
- [ ] Documentado em uma linha no README do projeto ou vault

## 12. Ver também

- skill `hermes-ops-producao`
- [instalacao-producao-vps.md](./instalacao-producao-vps.md)
- [profiles-e-kanban.md](./profiles-e-kanban.md)
- Docs: https://hermes-agent.nousresearch.com/docs/user-guide/features/cron
