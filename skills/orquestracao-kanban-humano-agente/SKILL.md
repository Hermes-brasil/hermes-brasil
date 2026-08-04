---
name: orquestracao-kanban-humano-agente
description: Orquestrar projetos com Kanban nativo do Hermes — humanos + agentes no mesmo board, profiles especialistas, pace control e pitfalls de produção.
---

# Orquestração Kanban — Humanos + Agentes de IA

> Conceito validado publicamente por co-fundadores da Nous (Hermes) e detalhado aqui com pitfalls de operação real no Kanban nativo do Hermes Agent.

Guia passo a passo: `guides/profiles-e-kanban.md` neste repositório.

## Princípio central

**UM só Kanban** para humanos e agentes. Mesmas colunas, mesmas tasks; executor intercambiável (pessoa ou profile de IA).

```text
"Posso ter um gerente de projeto humano no board manualmente
enquanto agentes preenchem as peças que ele pediu."
```

## Quando usar Kanban (vs delegate)

| Use Kanban | Use `delegate_task` |
|---|---|
| Trabalho que deve sobreviver a crash | Subtarefa curta na mesma sessão |
| Dependências entre etapas | Sem necessidade de auditoria |
| Gate humano (`blocked`) | Resposta única e volta |
| Vários especialistas / profiles | Um raciocínio paralelo leve |
| Audit trail importa | — |

## Colunas / status mentais

```text
Triage → todo → ready → in progress → blocked → done
```

No Hermes nativo os nomes de status variam por versão; o que importa:

- **`todo` é despachável** se não há parent pendente — não é “geladeira”
- **`blocked`** = estacionado de verdade (humano ou pace control)
- **`done`** desbloqueia filhos

## Arquitetura simples

```text
Orquestrador (profile default + gateway)
    → cria tasks com --assignee <profile>
Dispatcher (no gateway, ~60s)
    → promove quando parents done, claim, spawn hermes -p <profile>
Workers especialistas
    → orientam, trabalham, block ou complete com summary
```

```bash
hermes profile create writer --clone-from default
hermes kanban create "Redigir guia X" --assignee writer --body "Contexto + DoD"
hermes kanban list
hermes kanban dispatch   # se quiser tick manual
```

## Mecânica de orquestração

```text
ORQUESTRADOR
├── Quebra meta em tasks pequenas
├── Atribui a humanos ou profiles
├── Encadeia com --parent
├── Observa board (list/stats/show)
└── Escala blocked → humano

DURABILIDADE
├── Board persistente (SQLite do Hermes)
├── Handoff em summary/metadata
└── Trabalho não some com /new no chat
```

## Quem instrui quem

```text
Humano → agentes     (gerente define, workers executam)
Agente → humano      (block com pergunta objetiva)
Humano OU agente     na mesma etapa, conforme capacidade
```

## Pitfalls de produção (leia antes do bulk create)

1. **Assignee fantasma** — nome no board ≠ `hermes profile list`. Task fica ready para sempre.
2. **Rajada** — 20 tasks `todo`/`ready` = 20 spawns. Park em `blocked`, promova 1 a 1 se o modelo é caro.
3. **Reassign em massa** — corrigir 15 fantasmas já `ready` spawna 15. Bloqueie antes.
4. **`providers: {}` no profile** — apaga providers globais; fallback silencioso e 401 confuso.
5. **Skills só no default** — worker de profile pode não ver skill global; copie ou inline no body.
6. **Fallback premium em loop** — crash → modelo caro → conta sangra. Teto + fallback finito.
7. **claim_lock stale** após kill — limpe lock com backup do DB.
8. **Gateway cacheia board** — trocou board ativo → restart gateway.
9. **Publicar fora do board** sem olhar o board → duplicata com outro worker/cron.
10. **Tier dinheiro/publish/cliente** → `blocked`, não “fez e avisou”.

## Pace control (anti-burst)

Para pipelines de conteúdo ou qualquer fila cara:

- Máximo **1** task `ready|running` por assignee
- Resto `blocked` com motivo de pace
- Orquestrador/cron promove a próxima quando a atual completa

## Exemplos

```text
Conteúdo longo
├── researcher: outline
├── writer: capítulos (parent=outline)
├── humano: revisão de voz (blocked até OK)
└── publisher: só após aprovado

Ops
├── backend-dev: fix
├── reviewer: checklist
└── ops: deploy (gate humano se prod)
```

## Lifecycle do worker (resumo)

1. Ler a task por completo  
2. Executar no escopo (não re-decompor o mundo inteiro)  
3. Heartbeat se longo  
4. `complete` com summary acionável **ou** `block` com pergunta única e clara  

## Comandos úteis

```bash
hermes profile list
hermes kanban assignees
hermes kanban list
hermes kanban stats
hermes kanban show <id>
hermes kanban tail <id>    # se existir na versão
grep -i kanban ~/.hermes/logs/gateway.log | tail -30
```

Nota: `hermes doctor` pode avisar que kanban tools "não estão disponíveis" no shell interativo. Tools de worker só entram com `HERMES_KANBAN_TASK` no spawn. Isso não invalida o board. Confirme com `kanban list`.

## Ver também

- `guides/profiles-e-kanban.md`
- `guides/cron-em-producao.md`
- skill `hermes-ops-producao`
- Docs oficiais Hermes (Kanban + Profiles)
