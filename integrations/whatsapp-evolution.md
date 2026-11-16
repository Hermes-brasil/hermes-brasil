# WhatsApp → Hermes → WhatsApp (Evolution API)

Integração completa: uma mensagem chega no WhatsApp, o Hermes lê o texto, decide o que fazer e responde na mesma conversa.

Serve para: atendimento de primeiro contato, triagem de pedidos, resposta a dúvidas repetidas, aviso interno ("cliente X perguntou preço") e registro de conversa em planilha/banco.

> Testado com Evolution API **v2.3.7 / v2.4.0-rc2** (Docker) e Hermes Agent em Linux. Os passos marcados com ✅ foram executados de fato em produção; os demais são a configuração documentada de cada etapa. Onde o comportamento depende da versão, está dito.

---

## Antes de tudo: o risco é real

Evolution API fala WhatsApp pelo protocolo **não oficial** (Baileys, o mesmo do WhatsApp Web). Isso funciona, mas **não é uma API abençoada pela Meta** — e desde 2026 a Meta aperta a varredura contra clientes não oficiais. Números banidos por automação não oficiosa existem aos montes, inclusive com uma única mensagem enviada.

Regra de decisão:

| Cenário | Canal |
|---|---|
| Cliente pagante, número de negócio que não pode cair | **WhatsApp Business Cloud (API oficial)** |
| Uso próprio, prospecção, número descartável, teste | **Evolution API** (risco aceito) |
| Só receber e notificar (nunca responder) | Evolution API serve bem |

Se a resposta for "cliente pagante", este guia ainda ajuda na parte do Hermes — troque apenas a camada de transporte.

---

## O desenho

```
WhatsApp do cliente
      │  mensagem
      ▼
Evolution API  ──webhook──▶  relay (n8n ou script)
                                  │  texto limpo
                                  ▼
                            Hermes (agente + skills)
                                  │  resposta
                                  ▼
                     POST /message/sendText/{instancia}
                                  │
                                  ▼
                        WhatsApp do cliente
```

Duas decisões antes de começar:

1. **Quem filtra o payload?** A Evolution manda um JSON grande, com evento, instância, chave, tipo de mensagem e mídia em base64. Não mande isso cru para o agente: extraia `numero`, `texto` e `fromMe` e descarte o resto. Payload cru queima contexto e faz o modelo responder ao JSON.
2. **Quem responde?** Se o agente responde, o relay **não** pode responder também — senão o cliente recebe duas mensagens.

---

## Passo 1 — Conectar a instância

Crie com QR Code. Sem `number` no corpo (com `number` a v2 responde "Invalid integration"):

```bash
curl -s -X POST "$EVO/instance/create" \
  -H "apikey: $EVO_KEY" -H "Content-Type: application/json" \
  -d '{"instanceName":"atendimento","integration":"WHATSAPP-BAILEYS","qrcode":true}'
```

A resposta traz `qrcode.base64` (PNG em data-URL). Se a instância já existe e caiu, **não recrie** — peça o QR de novo:

```bash
curl -s "$EVO/instance/connect/atendimento" -H "apikey: $EVO_KEY"
```

✅ Na v2.3.7 esse endpoint funciona com **GET**; o POST documentado retorna 404. `pairingCode` pode vir `null` mesmo com QR válido — nesse caso conecte por QR e siga.

### Verificar que a conexão é REAL

Este é o passo que quase todo mundo pula, e é onde nasce a maior parte dos "a API diz que enviou mas não chegou":

```bash
curl -s "$EVO/instance/connectionState/atendimento" -H "apikey: $EVO_KEY"
curl -s "$EVO/instance/fetchInstances" -H "apikey: $EVO_KEY"
```

✅ `connectionStatus: "open"` **não** é prova de nada. O sinal de conexão real é `profileName` preenchido (e `ownerJid` com o número correto):

```json
{
  "name": "atendimento",
  "connectionStatus": "open",
  "profileName": "Seu Nome",        ← se for null, a conexão é falsa
  "ownerJid": "5511999999999@s.whatsapp.net"
}
```

`open` + `profileName: null` = QR aceito, pareamento não completou. Saída: deletar e recriar a instância com QR novo.

---

## Passo 2 — Apontar o webhook para o seu relay

```bash
curl -s -X POST "$EVO/webhook/set/atendimento" \
  -H "apikey: $EVO_KEY" -H "Content-Type: application/json" \
  -d '{
    "webhook": {
      "url": "https://SEU_DOMINIO/webhook/whatsapp",
      "enabled": true,
      "events": ["MESSAGES_UPSERT"]
    }
  }'
```

✅ Três detalhes que quebram silenciosamente:

- O **wrapper `webhook`** é obrigatório — o corpo sem ele é aceito e ignorado.
- `events` usa o enum da versão: `MESSAGES_UPSERT` (não `messages.upsert`).
- Confira o que ficou salvo, sempre: `GET $EVO/webhook/find/atendimento`. A configuração persiste no banco da Evolution e sobrevive a restart do container.

Se a Evolution estiver em Docker Swarm sem porta publicada, `localhost` não existe para ela: use o nome do serviço na rede Docker ou uma URL pública.

---

## Passo 3 — Extrair só o que importa

Formato de referência do `MESSAGES_UPSERT` na v2 (confira no seu primeiro payload antes de confiar — a versão muda detalhes):

```json
{
  "event": "messages.upsert",
  "instance": "atendimento",
  "data": {
    "key": {
      "remoteJid": "5511988887777@s.whatsapp.net",
      "fromMe": false,
      "id": "ABCDEF..."
    },
    "pushName": "Maria",
    "message": {
      "conversation": "quanto custa?"
    }
  }
}
```

Regras de filtro, em ordem — todas já custaram um incidente a alguém:

- **`fromMe: true` → descarte.** Sem isso o bot responde a si mesmo em loop infinito.
- **`remoteJid` terminando em `@g.us` → grupo.** Descarte, ou o bot fala em grupo sem permissão.
- **`remoteJid == "status@broadcast"` → status.** Descarte.
- **Áudio/imagem/documento:** o texto vem vazio. `message.conversation` não existe; o conteúdo está em `message.extendedTextMessage.text`, `message.audioMessage`, `message.imageMessage`. Baixe por `POST /chat/getBase64FromMediaMessage/{instancia}` e transcreva antes de chamar o agente — ou responda "recebi seu áudio, já te respondo" e trate à parte.
- **Idempotência:** guarde `data.key.id`. A Evolution pode reenviar o mesmo evento, e webhook sem dedupe vira resposta duplicada.

O texto limpo que vai para o agente:

```
Nome: Maria
Mensagem: quanto custa?
```

---

## Passo 4 — Hermes decide e responde

Duas formas de acionar o agente:

**A) one-shot por CLI** (relay e Hermes na mesma máquina):

```bash
hermes -z "Você atende a loja X. Responda em português, curto. Cliente Maria perguntou: quanto custa?"
```

✅ Só a resposta sai no stdout (o banner de aviso vai no stderr). Capture stdout limpo, nunca `2>&1`.

**B) plataforma `webhook` do gateway** (relay em container/outra máquina, com HMAC e idempotência): ver [n8n → Hermes](./n8n-hermes.md), caminho B — a assinatura e o formato são os mesmos, só muda quem é o emissor.

Depois de obter a resposta, envie:

```bash
curl -s -X POST "$EVO/message/sendText/atendimento" \
  -H "apikey: $EVO_KEY" -H "Content-Type: application/json" \
  -d '{"number":"5511988887777","text":"Custa R$ 80,00. Quer que eu reserve?","options":{"delay":1200}}'
```

- `number` é o `remoteJid` **sem** o `@s.whatsapp.net`.
- Nome de instância com espaço precisa de URL-encode (`hermes%20meu%20celular`) — senão: `URL can't contain control characters`. Prefira nome com hífen.
- ✅ HTTP 400 com `exists: false` = o número não tem WhatsApp. Não retente, marque como inválido.
- ✅ HTTP 500 "Connection Closed" = instância caiu. Verifique `connectionState` e reconecte antes de reenviar.

---

## Passo 5 — Verificação antes de confiar

Na ordem, sem pular:

1. Mande mensagem **do seu próprio número para a instância** e veja se o webhook chega (log do relay).
2. Veja se a resposta chega no seu WhatsApp. ✅ A mensagem enviada **aparece na conversa** no WhatsApp do número pareado — se não aparecer, o envio não aconteceu.
3. Teste com um segundo número real (o seu outro chip). Mensagem para si mesmo sempre funciona e **esconde** o problema de roteamento.
4. Deixe rodando 24h e conte quantas respostas vieram sem você tocar.

---

## Pitfalls (todos observados em produção)

- **`status: PENDING` não é entrega.** A API responde 200 com PENDING e a mensagem nunca chega ao destinatário externo — sintoma clássico depois de reconexão. Confirme no aparelho, não no JSON.
- **`device_removed` (401)** derruba a sessão. Reconectar é manual (QR). Automatize o alerta, não o pareamento.
- **Não reconecte mais de 2x no mesmo dia.** ✅ Três ou mais reconexões no mesmo dia e o WhatsApp passa a aceitar envio (`PENDING`) sem entregar nada, por 24-48h.
- **Aqueça o número.** Número novo + volume = ban rápido. Comece com conversas reais por alguns dias antes de qualquer automação.
- **Delay de 2 a 5 minutos entre mensagens de prospecção**; 1,2s só para resposta de atendimento em conversa já iniciada.
- **Nunca verifique números em massa** (`/chat/whatsappNumbers`). Checar se existe WhatsApp já acende alerta de spam.
- **Se mais de 5% dos contatos bloquearem, pare a campanha.** É o único sinal que a Meta dá antes do ban.
- **Webhook não tem retry.** Se o relay estiver fora do ar na hora do evento, a mensagem se perde. Para conversa de cliente, isso é perda de venda — use fila (Redis) ou aceite e monitore.
- **Webhook do relay sem autenticação** é porta aberta para qualquer um injetar mensagem no seu agente. Use header com segredo e valide no relay.

---

## Checklist

- [ ] `connectionState` = `open` **e** `profileName` preenchido
- [ ] `GET /webhook/find/{instancia}` mostra `enabled: true` e `MESSAGES_UPSERT`
- [ ] relay descarta `fromMe`, `@g.us` e `status@broadcast`
- [ ] relay deduplica por `data.key.id`
- [ ] resposta chega em número externo (não no próprio)
- [ ] segredo no header do webhook e validado no relay
- [ ] alerta quando `connectionState` virar `close`
- [ ] se o número é de cliente pagante: use a API oficial da Meta
