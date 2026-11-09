# n8n → Hermes → Telegram

Integração completa: um evento no n8n dispara o Hermes, o agente executa a tarefa e a resposta chega no seu Telegram.

Serve para: resumir e-mail que chegou, responder formulário, processar planilha, monitorar site e avisar, transformar mensagem de cliente em rascunho de resposta.

> Testado em Hermes Agent (canal stable, Linux) com n8n 2.x em Docker Swarm. Os comandos marcados com ✅ foram executados de fato; os demais são a configuração documentada de cada etapa.

---

## O desenho

```
n8n (Webhook / Cron / IMAP)
        │  POST JSON
        ▼
  ┌───────────────┐        ┌──────────────────┐
  │ rota do       │───────▶│ agente Hermes     │
  │ Hermes        │        │ + skills          │
  └───────────────┘        └────────┬─────────┘
                                    │ resposta
                                    ▼
                            Telegram (chat definido)
```

Duas decisões antes de começar:

1. **Onde o n8n roda?** Se está em container, ele não vê o `hermes` do host. Isso decide o caminho (A ou B) — é o erro nº 1 de quem monta esse fluxo.
2. **Quem entrega a mensagem?** O Hermes entrega direto no Telegram (recomendado) ou o n8n devolve no node response e entrega. Escolha um só, senão a mensagem chega duplicada.

---

## Caminho A — `hermes -z` (one-shot por CLI)

O jeito mais simples: o n8n executa um comando e recebe o texto de volta.

### A1. Verificar que o one-shot funciona

```bash
hermes -z "Responda apenas: OK"
```

✅ Saída observada (só isto no stdout):

```
OK
```

Três coisas que você precisa saber sobre esse comando, todas verificadas:

- **Ele é não-interativo**: roda um turno, imprime a resposta e sai com código `0`. Não abre sessão de chat.
- **Só a resposta vai para o stdout.** O banner de avisos (`Claude Subscription DirectSDK...`) sai no **stderr**. Se você capturar `2>&1`, o parse quebra. Capture stdout limpo.
- **`-z` não mantém contexto entre chamadas.** Cada POST é uma conversa nova. Se precisar de memória entre eventos, use `hermes --continue` com sessão nomeada ou o caminho B.

Para fixar modelo e reduzir custo:

```bash
hermes -z "Classifique este ticket em uma palavra: ${TEXTO}" -m SEU_MODELO_ECONOMICO
```

### A2. n8n no mesmo host (systemd / processo local)

Node **Execute Command**:

```bash
hermes -z "Resuma em 5 linhas o texto a seguir e responda em português: {{ $json.body.texto }}"
```

O stdout do node é a resposta do agente. Ligue direto num node Telegram (ou devolva no `Respond to Webhook`).

### A3. n8n em Docker → pitfall que quebra o fluxo

Se o n8n roda em container, o node Execute Command executa **dentro do container**, onde não existe `hermes`, nem a venv, nem `~/.hermes`. O sintoma é sempre o mesmo:

```
sh: 1: hermes: not found
```

Três saídas, em ordem de preferência:

**1. SSH do container para o host** (funciona sem mexer em nada):

```bash
ssh hermes@HOST 'hermes -z "Resuma: {{ $json.body.texto }}"'
```

Requisitos: chave SSH do container para o host, `HOME` correto no comando remoto. Se o SSH reclamar de `~/.hermes` inexistente, force `HOME`:

```bash
ssh hermes@HOST 'HOME=/home/hermes hermes -z "..."'
```

**2. Compartilhar o binário no container** (mais frágil, exige montar venv + config):

```
-v /usr/local/lib/hermes-agent:/opt/hermes:ro
-e HOME=/hermes-home
```

Só vale se você não tiver SSH — o Hermes depende de `~/.hermes` gravável (sessões, skills, cache).

**3. Usar o caminho B** (webhook HTTP). É o que eu recomendo para n8n em container: não depende de nada do lado do host além de uma porta.

### A4. Entregar no Telegram

Duas opções — **não use as duas ao mesmo tempo**:

**(a) O n8n entrega.** A resposta volta no node e você usa um node Telegram com `chatId` fixo. Simples, e o custo é zero de gateway.

**(b) O Hermes entrega.** O agente responde e você envia pelo próprio Hermes:

```bash
hermes send -t telegram "✅ Tarefa concluída: {{ $json.resposta }}"
```

`hermes send` **não usa modelo e não precisa do gateway rodando** para Telegram/Discord/Slack — ele reaproveita as credenciais do `~/.hermes/.env`. Ótimo para alertas de custo zero.

Para descobrir o destino correto:

```bash
hermes send --list
hermes send --list telegram
```

✅ Saída observada:

```
Telegram:
  telegram:Danilo Almeida (dm)
```

Uso com destino explícito e silencioso em script:

```bash
hermes send -t telegram:SEU_CHAT_ID -q "backup concluído"
```

### A5. Verificação do caminho A

1. Rode o comando à mão no host: tem que imprimir a resposta.
2. Rode o mesmo comando **de dentro do container**: `docker exec -it N8N sh -c 'hermes --version'`. Se falhar, você está no A3.
3. Dispare o workflow e confira que a mensagem chegou **uma** vez.

---

## Caminho B — plataforma `webhook` (HTTP assinado)

Quando o n8n não tem acesso ao host, ou quando você quer HMAC, idempotência e filtro por evento. A porta é aberta pelo gateway, e cada rota tem nome e segredo próprios.

### B1. Habilitar a plataforma

```bash
hermes webhook list
```

Se a plataforma não estiver ligada, o próprio comando imprime as instruções. O caminho sem editar YAML é o `.env`:

```bash
WEBHOOK_ENABLED=true
WEBHOOK_PORT=8644
WEBHOOK_SECRET=SEU_SEGREDO_GLOBAL
```

Substitua o `SEU_SEGREDO_GLOBAL` por algo gerado (`openssl rand -hex 32`) e reinicie o gateway:

```bash
hermes gateway install   # uma vez, se ainda não é serviço
hermes gateway run       # ou reinicie o serviço existente
```

Confira que subiu:

```bash
curl -s http://127.0.0.1:8644/health
```

Resposta esperada: `{"status": "ok", "platform": "webhook"}`.

### B2. Criar a rota

```bash
hermes webhook subscribe resumo-email \
  --prompt "Resuma o e-mail abaixo em 5 linhas, em português, e diga se exige resposta hoje:\n\nAssunto: {subject}\nDe: {from}\n{body}" \
  --events email.received \
  --deliver telegram \
  --deliver-chat-id SEU_CHAT_ID \
  --description "Resumo diário de e-mails que chegam"
```

O que cada peça faz:

- **`resumo-email`** é o nome na URL: `POST /webhooks/resumo-email`.
- **`--prompt`** usa `{dot.notation}` para puxar campos do JSON recebido. É aqui que o payload vira instrução.
- **`--events`** filtra por `event_type`; POST de outro tipo volta `ignored` e não gasta modelo.
- **`--deliver` + `--deliver-chat-id`** decidem onde a resposta aparece. `log` só grava, sem enviar.
- **`--secret`** é gerado automaticamente se você omitir. Ele é o HMAC da rota.

Extras úteis:

| Flag | Para quê |
|---|---|
| `--deliver-only` | não chama o modelo: só renderiza o prompt e entrega (custo zero) |
| `--script nome.py` | filtra o payload em `~/.hermes/scripts/`; `[SILENT]` descarta o evento |
| `--skills a,b` | carrega skills específicas para essa rota |
| `--cron-job NOME` | dispara um job existente em vez de um turno novo |
| `--mirror-to-session` | grava a entrega na sessão do chat, para você responder com contexto |

### B3. Testar a rota antes de ligar no n8n

```bash
hermes webhook list
hermes webhook test resumo-email --payload '{"event_type":"email.received","subject":"Nota fiscal","from":"fin@x.com","body":"Segue a NF de outubro."}'
```

O teste assina o corpo com o segredo da rota e usa exatamente os headers que o n8n precisa replicar:

```
Content-Type: application/json
X-Hub-Signature-256: sha256=<hmac_sha256_hex(secret, body)>
X-GitHub-Event: <event_type>
```

A resposta do gateway **não é a resposta do agente** — o processamento é assíncrono:

```json
{"status": "accepted", "route": "resumo-email", "event": "email.received"}
```

Outros retornos que você vai ver e o que significam:

| Resposta | Significado |
|---|---|
| `accepted` | aceito, o agente vai rodar e entregar no canal |
| `ignored` | `event_type` fora do filtro, ou `--script` descartou |
| `duplicate` | mesmo `delivery_id` já processado (idempotência) |
| `error: unauthorized` | HMAC errado — veja B4 |

### B4. Gerar o HMAC no n8n

O corpo precisa ser assinado **exatamente** como será enviado (mesmos bytes, sem reindentar o JSON). Node **Code**:

```javascript
const crypto = require('crypto');
const corpo = JSON.stringify({ event_type: 'email.received', subject: $json.subject, from: $json.from, body: $json.body });
const assinatura = 'sha256=' + crypto.createHmac('sha256', 'SEU_SEGREDO_DA_ROTA').update(corpo).digest('hex');
return [{ json: { corpo, assinatura } }];
```

Node **HTTP Request**:

```
Method: POST
URL: http://HOST:8644/webhooks/resumo-email
Headers:
  Content-Type: application/json
  X-Hub-Signature-256: {{ $json.assinatura }}
Body: {{ $json.corpo }}   (raw/JSON, não deixe o n8n re-serializar)
```

Se der `unauthorized` mesmo com o segredo certo, o culpado é quase sempre re-serialização do JSON pelo n8n (ordem de chaves ou espaços). Mande o corpo como texto bruto.

### B5. Verificação do caminho B

1. `curl /health` responde `ok`.
2. `hermes webhook test` devolve `accepted` e a mensagem chega no Telegram **uma** vez.
3. `hermes webhook test` com payload de `event_type` diferente devolve `ignored` e não gasta modelo.
4. Só então ligue o n8n.

---

## Checklist de produção

- [ ] Porta do webhook **não** exposta na internet sem proxy com TLS (Traefik/nginx na frente).
- [ ] Segredo por rota, não global, quando há mais de uma integração.
- [ ] Se o n8n pode reenviar o mesmo evento, use `delivery_id` para a idempotência funcionar.
- [ ] Uma única camada entrega no Telegram (agente **ou** n8n).
- [ ] Timeout do n8n maior que o pior tempo do agente, ou processamento assíncrono.
- [ ] Se o fluxo grava arquivo, o `HOME` do processo precisa ser o do usuário que tem `~/.hermes`.

## Erros comuns

| Sintoma | Causa |
|---|---|
| `hermes: not found` | n8n em container sem o binário (A3) |
| resposta com banner junto | capturou `2>&1`; use só stdout |
| agente "não lembra" do evento anterior | `-z` não mantém sessão; use rota com `--mirror-to-session` |
| mensagem duplicada | agente entrega **e** n8n entrega |
| `unauthorized` | HMAC sobre JSON re-serializado, ou segredo de outra rota |
| funciona à mão, falha no cron | `HOME`/`PATH` diferentes no ambiente do serviço |
