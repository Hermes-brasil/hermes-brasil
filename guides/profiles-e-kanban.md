# Profiles + Kanban: multi-agente com Hermes

Como rodar **um orquestrador** + **especialistas** sem virar zoo de processos soltos. Padrão alinhado ao Kanban nativo do Hermes e ao que a comunidade chama de “um board para humanos e agentes”.

## Quando usar o quê

| Situação | Ferramenta |
|---|---|
| Subtarefa rápida na mesma sessão | `delegate_task` |
| Trabalho que sobrevive a crash, com dependência e auditoria | **Kanban** |
| Especialista com modelo/persona/skills próprios | **Profile** + assignee Kanban |
| Só um chat contínuo | Profile default + gateway |

Regra: se precisa de handoff, fila, blocked humano ou paralelismo com rastro — Kanban. Se é “pensa 2 minutos e volta” — delegate.

## Arquitetura recomendada (A)

```text
┌─────────────────────────────┐
│  Profile default            │
│  (orquestrador + gateway)   │
│  cron · inbox · decomposição│
└─────────────┬───────────────┘
              │ hermes kanban create --assignee X
              ▼
┌─────────────────────────────┐
│  Dispatcher (no gateway)    │
│  todo→ready quando parents  │
│  claim atômico + spawn -p   │
└─────────────┬───────────────┘
              ▼
     profiles especialistas
     (writer, backend-dev, seo-expert, ops…)
```

- Orquestrador roda 24/7 no gateway principal
- Especialistas sobem **on-demand** com `hermes -p <profile>`
- Cada profile tem `config.yaml`, `.env`, `SOUL.md` e skills isoláveis

## 1. Criar profiles

```bash
# herda skills/config do default
hermes profile create writer --clone-from default
hermes profile create backend-dev --clone-from default
hermes profile create seo-expert --clone-from default

hermes profile list
hermes profile show writer
```

Configure modelo por profile:

```bash
hermes config set model.default "seu-modelo" --profile writer
hermes config set model.provider "seu-provider" --profile writer
```

### Pitfall: `providers: {}` silencioso

Se o `config.yaml` do profile tem `providers: {}`, isso **apaga** as definições globais. O modelo “configurado” cai no fallback e você debuga 401 no endpoint errado.

Sempre declare o provider primário no profile:

```yaml
providers:
  meu-provider:
    base_url: https://api.exemplo.com/v1
    api_mode: chat_completions
    api_key_env: MINHA_API_KEY
    default_model: modelo-x
```

E garanta que a chave está **descomentada** no `.env` do profile (clone deixa template comentado).

### SOUL.md

Persona curta em `~/.hermes/profiles/<nome>/SOUL.md`:

- identidade e competência
- o que NÃO faz sem aprovação
- formato de saída
- idioma

Menos romance, mais contrato operacional.

## 2. Kanban básico

```bash
hermes kanban init
hermes kanban create "Título da task" \
  --assignee writer \
  --body "Contexto completo. Critérios de pronto. Caminhos."
hermes kanban list
hermes kanban stats
hermes kanban show <id>
```

Dependências:

```bash
hermes kanban create "Pesquisa" --assignee researcher --body "..."
# anote o id, depois:
hermes kanban create "Redigir" --assignee writer --body "..." --parent <id-pesquisa>
```

Dispatcher embutido no gateway (config típica `kanban.dispatch_in_gateway: true`) a cada ~60s:

- promove `todo` → `ready` quando parents estão `done`
- faz claim e spawna worker

## 3. Colunas mentais (humano + agente)

Mesmo board, papéis intercambiáveis:

| Coluna / status | Significado |
|---|---|
| Triage / ideias | ainda não é task |
| todo | na fila; **é despachável** se sem parent pendente |
| ready | claimable |
| in progress / running | executor ativo |
| blocked | precisa humano ou gate |
| done | pronto; desbloqueia filhos |

**Pitfall #1:** `todo` **é** despachável. Para estacionar, use `blocked`, não “deixar em todo e rezar”.

## 4. Lifecycle do worker

O Hermes injeta orientação de worker automaticamente. Em resumo o especialista deve:

1. Orientar-se (`kanban show` / contexto da task)
2. Trabalhar no escopo
3. Heartbeat se longo
4. `block` com pergunta clara **ou** `complete` com summary + metadata úteis

Handoff bom = próximo humano/agente consegue seguir sem reabrir o chat original.

## 5. Pitfalls de produção (os que doem)

1. **Assignee fantasma** — `backend-eng` no board mas profile real é `backend-dev`. Task fica `ready` para sempre. Antes de bulk create: `hermes profile list` e `hermes kanban assignees`.

2. **Rajada de spawn** — criar 20 tasks `todo`/`ready` de uma vez = 20 workers. Para backlog grande: crie como `blocked` e promova 1 por vez (pace control).

3. **Reassign em massa** — corrigir 15 assignees fantasmas que já estão `ready` spawna 15 de uma vez. Park (`blocked`) primeiro; libere uma.

4. **Skills do sistema vs do profile** — worker vê sobretudo o que está no profile. Instrua no body da task ou copie a skill para o profile.

5. **Fallback caro em loop** — worker crasha → fallback premium → conta sangra. Teto de custo + fallback barato e finito.

6. **claim_lock stale** — task presa após kill brutal. Limpe lock/expiry via SQLite com cuidado (backup antes).

7. **Board ativo em cache** — gateway cacheia board no start; trocou de board → restart gateway. Scripts devem revalidar board ativo.

8. **dry-run que promove** — não use dry-run como “só olhar” em fila parked; confira estado depois.

9. **Conteúdo duplicado** — se publica fora do board, confira o board antes (race com outro worker/cron).

10. **Humano no loop** — dinheiro, publish em nome da pessoa, schema de prod, contato com cliente: `blocked` + pergunta, não “agiu e avisou”.

## 6. Padrões úteis

### Fan-out + fan-in

N researchers em paralelo (sem parent) → 1 analyst com todos como parents.

### Pipeline com gate

`pm → backend-dev → reviewer` encadeado por `--parent`. Reviewer pode `block` pedindo decisão humana.

### Pace control (conteúdo / tokens)

Máximo **1** task `ready|running` por assignee caro. Resto `blocked` com `block_kind` de pace. Cron ou orquestrador promove a próxima.

### Um board só

Humanos movem cards / comentam a mesma verdade que os agentes. Dois sistemas (Trello pra gente + JSON pra IA) divergem em uma semana.

## 7. Observabilidade sem dashboard

```bash
hermes kanban list
hermes kanban stats
hermes kanban show <id>
hermes kanban tail <id>     # se disponível na versão
# logs
grep -i kanban ~/.hermes/logs/gateway.log | tail -40
```

`hermes doctor` pode marcar kanban como “dependency not met” porque tools de worker só liberam com `HERMES_KANBAN_TASK` no spawn — isso **não** significa que o board está morto. Valide com `kanban list`.

## 8. Smoke test de profile novo

```bash
hermes kanban create "Smoke: profile writer" \
  --assignee writer \
  --body "Responda com: nome do profile, modelo ativo, e 'ok'. Nada mais."
# aguarde dispatcher ou:
hermes kanban dispatch
hermes kanban show <id>
```

## 9. Ver também

- skill da comunidade: `orquestracao-kanban-humano-agente` (conceitos + pitfalls)
- skill `hermes-ops-producao`
- [cron-em-producao.md](./cron-em-producao.md)
- Docs: https://hermes-agent.nousresearch.com/docs/ (Kanban + Profiles)
