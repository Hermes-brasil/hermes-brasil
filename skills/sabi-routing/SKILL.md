---
name: sabi-routing
description: Roteie cada inferência do Hermes pelo Sabi (alias sabi-code) — use quando quiser trocar cheap/mid/strong por round sem forkar o loop do Hermes.
---

# Sabi Routing no Hermes

Sabi (https://github.com/vizuh/sabi) é um agendador de inferência: o Hermes mantém o loop nativo e o Sabi decide modelo/esforço/provider por request, atrás do alias `sabi-code`.

Caminho V1 suportado:

```text
Hermes → custom:sabi / sabi-code → proxy Sabi 127.0.0.1:8787 → upstream
```

## Quando carregar

- Quer rotear `sabi-code` por trajetória (tool-result, falha, retomada) em vez de um modelo fixo
- Vai operar Hermes + Sabi com `HERMES_HOME` isolado
- Precisa de rollback rápido para o perfil Hermes original

Não use para: transferir assinatura (Nous / OpenCode Go / ChatGPT Plus) para outro provider, impor orçamento/permissão (middleware é fail-open), ou cobrir subagentes/compaction/gateway (não verificado).

## Pré-requisitos

- Hermes `0.21.3` (commit `01382698`, `NousResearch/hermes-agent`). Verifique: `hermes --version`
- Checkout Sabi com `npm install` feito
- Perfil Hermes **novo e vazio** — nunca instale no `~/.hermes` pessoal
- Modelo custom no Hermes exige contexto ≥ 64000 (ex.: Neste host `qwen2.5-coder:7b` tem 32768 e é rejeitado; use Nous ou local ≥64k)

## Setup — rota Nous (padrão)

```bash
npm run setup -- --harness=hermes --hermes-home="$HOME/.config/sabi/hermes" --no-jev
export HERMES_HOME="$HOME/.config/sabi/hermes"
hermes auth add nous --type oauth

# Terminal 1 — proxy Nous do Hermes
HERMES_HOME="$HERMES_HOME" hermes proxy start --provider nous --host 127.0.0.1 --port 8645

# Terminal 2 — proxy Sabi
SABI_CONFIG="$HERMES_HOME/sabi.config.json" npm start

# Terminal 3 — chat
export SABI_HERMES_BASE_URL="http://127.0.0.1:8787/v1"
HERMES_HOME="$HERMES_HOME" SABI_HERMES_BASE_URL="$SABI_HERMES_BASE_URL" hermes chat
```

## Setup — OpenRouter uma chave (sem login Nous)

```bash
npm run setup -- --harness=hermes --upstream=openrouter \
  --hermes-home="$HOME/.config/sabi/hermes" --no-jev

SABI_CONFIG="$HOME/.config/sabi/hermes/sabi.config.json" npm start
HERMES_HOME="$HOME/.config/sabi/hermes" hermes chat
```

Só pede `OPENROUTER_API_KEY` (TTY oculto). BYOK/prioridade/fallback ficam no OpenRouter. Sabi nunca recebe chaves de assinatura.

## Como funciona

- Plugin `llm_request` (zero-dependência) adiciona só `X-Sabi-Client/Session/Turn` no endpoint loopback exato com model `sabi-code`
- Copia o request completo: tools, ids, ordem, argumentos, streaming. Não reescreve provider, não faz retry, não cria segundo loop
- `turn_id` do Hermes agrupa um user-turn + tool-loop; Sabi cria request-ID próprio por inferência e decide por trajetória
- Falha do middleware = fail-open (request segue sem atribuição)

## Verificação observável

1. No chat com `sabi-code`, peça uma tarefa com leitura de arquivo + follow-up
2. No log do Sabi confira `client: hermes`, `sessionKnown: true`, request-IDs únicos por inferência
3. Retome a sessão — deve gerar novo `turn_id` com mesma sessão
4. `hermes --version` + base `SABI_HERMES_BASE_URL` loopback exata; outro host = request intocado (by design)

## Rollback

```bash
# voltar ao perfil original: pare de selecionar o HERMES_HOME isolado,
# ou no perfil isolado:
# remova `sabi-metadata` de `plugins.enabled`
```

Preserve o perfil. Não apague state do usuário nem mude outros providers.

## Limites explícitos (não omitir no PR)

- Fixado/testado em Hermes `0.21.3` / `01382698`; outra versão = revalidar
- `discover_models: false` não suprime todos os GETs de metadata local (observado: 404s não quebram a tarefa)
- Cobertura: conversa principal + resume com tool-loop limitado. Subagentes, compaction, MoA, gateway, desktop: não exercidos
- Smoke pago / produção / economia medida: gates separados, não afirmados aqui
- Sem segredos no repo: credencial fica no auth store do Hermes ou no ambiente do Sabi

## Referências

- Sabi: https://github.com/vizuh/sabi — `docs/adapters/hermes.pt-BR.md`, `packages/adapters/hermes/README.md`
- Evidência: `docs/research/hermes-compatibility.md` (12 testes do adapter, probe nativo 3 requests, `mid → cheap → mid` via Sabi, 1 smoke Nous `NOUS_SABI_OK`)
- Hermes oficial: https://github.com/NousResearch/hermes-agent @ `01382698`
