# Integrações ponta a ponta

Cada arquivo aqui é uma integração **completa**: trigger → agente → entrega, com passo de verificação antes de você confiar no fluxo.

O padrão é sempre o mesmo, e é o que evita a maior parte dos bugs:

```
[ evento ]  →  [ Hermes decide/executa ]  →  [ resultado no canal ]
   n8n           agente + skills              Telegram/Slack/e-mail
webhook, cron,
formulário,
e-mail, planilha
```

A regra de ouro: **o evento não fala com o agente; ele fala com uma rota**. A rota tem nome, segredo e destino de entrega. Sem isso, o fluxo depende de alguém lembrar de onde veio o POST.

## Integrações disponíveis

- [n8n → Hermes → Telegram](./n8n-hermes.md) — webhook do n8n dispara o agente e a resposta chega no Telegram. Inclui os dois caminhos (CLI one-shot e webhook assinado) e o pitfall de n8n em container.

## Qual caminho usar

| Situação | Caminho |
|---|---|
| n8n e Hermes na **mesma máquina**, execução rápida (< 2 min) | `hermes -z` por CLI |
| n8n em **Docker/outra máquina**, ou precisa de HMAC e idempotência | plataforma `webhook` do gateway |
| só quer **notificar** (sem gastar modelo) | `hermes send` |
| quer um **job agendado** com histórico e ledger | `hermes cron` |

## Antes de abrir um PR

- Cole no arquivo o **comando de verificação** de cada etapa (o que você rodou e o que voltou).
- Diga qual versão do Hermes e do n8n você testou.
- Nada de segredo, token ou chat_id real no arquivo — use `SEU_SEGREDO`, `SEU_CHAT_ID`.
