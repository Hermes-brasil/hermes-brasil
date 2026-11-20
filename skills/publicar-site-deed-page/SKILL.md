---
name: publicar-site-deed-page
description: Use quando o usuário pedir para publicar, hospedar ou gerar um link público para uma página HTML ou pasta de site estático. Publica no deed.page só após prévia e um "sim" explícito.
version: 1.0.0
author: Avatar 8 LLC (equipe deed.page)
license: MIT
metadata:
  hermes:
    tags: [publicacao, hospedagem, site-estatico, html, deed-page, pt-br]
---

# Publicar site estático no deed.page

O [deed.page](https://deed.page) hospeda arquivos estáticos (HTML, CSS, JS, imagens) em `https://<slug>.deed.page/` com uma requisição HTTP. Não precisa de conta nem de chave de API: a primeira publicação devolve um token de dono, que é a única chave para atualizar ou apagar o site. Skill oficial e completa (em inglês): https://github.com/f-o-x11/deed-page-skill

**O site é público.** Qualquer pessoa com o link vê os arquivos. Por isso esta skill nunca publica sem prévia e confirmação.

## Quando carregar

- O usuário pede "publica", "coloca no ar", "hospeda", "me dá um link" para um relatório HTML, dashboard, landing page, protótipo, slides ou pasta de docs gerada.
- O usuário quer atualizar ou apagar um site deed.page que ele já publicou nesta máquina.

Não use para: sites com backend, banco de dados ou login (o deed.page serve só arquivos estáticos); conteúdo com dados pessoais, segredos ou qualquer coisa que não deva ser pública; pedidos só de criar ("faz um site") — nesse caso gere os arquivos localmente e ofereça publicar.

## Regras (não pule)

1. **Prévia + "sim" explícito antes de todo upload.** Mostre ao usuário a URL pública de destino e a lista exata de arquivos (passo 2). Só publique depois de uma confirmação explícita ("sim", "pode publicar"). Silêncio, "ok, depois vemos" ou ausência de resposta não é confirmação.
2. **Execuções sem humano (cron, gateway, jobs) não publicam**, a menos que a instrução daquele job autorize publicar no deed.page de forma explícita. Na dúvida, pare e registre a prévia para o usuário aprovar.
3. **O token nunca entra na conversa.** Não rode `cat` no arquivo do token, não ecoe `$TOKEN`, não coloque o token em arquivos publicados, commits, logs ou memória. Os comandos abaixo leem o token dentro do shell e mascaram a resposta.
4. **Publique só a pasta de saída limpa** (`dist/`, `build/`, `out/`, `public/`, `_site/`), nunca a raiz do projeto, um checkout git ou `node_modules`.
5. **Planos pagos:** se a API devolver erro de limite com link de upgrade, mostre o `user_message` ao usuário como veio e não tente de novo. Nunca pague nada sem aprovação explícita do usuário.

## Procedimento

### 1. Escolha a pasta e o slug

```bash
DIR=./dist                         # pasta de saída limpa; precisa ter index.html
test -f "$DIR/index.html" || echo "ERRO: falta $DIR/index.html"
SLUG="relatorio-$(head -c4 /dev/urandom | od -An -tx1 | tr -d ' \n')"
```

Slug: minúsculas, dígitos e hífen, 1 a 63 caracteres, sem hífen no início ou fim. O sufixo aleatório evita colisão (`hello`, `my-page`, `my-app` e `my-site` são reservados).

### 2. Monte o pacote e mostre a prévia

```bash
TMP="$(mktemp -d)"; LISTA="$TMP/arquivos.txt"; ARQ="$TMP/site.tgz"
(cd "$DIR" && find . -type f ! -path '*/.*' ! -path '*/node_modules/*' \
  ! -name '*.pem' ! -name '*.key' ! -name 'id_rsa*' ! -name 'credentials.json' ! -name 'secrets.*' \
  | sed 's|^\./||' | sort) > "$LISTA"
tar -C "$DIR" -czf "$ARQ" -T "$LISTA"
tar -tzf "$ARQ"
du -h "$ARQ"
```

O `find` deixa de fora dotfiles e pastas ocultas (`.env`, `.git`, `.npmrc`, `.DS_Store`...), `node_modules`, chaves e links simbólicos. A lista impressa por `tar -tzf` é exatamente o que será enviado. Mostre ao usuário:

> Vou publicar estes N arquivos em **https://SLUG.deed.page/** (público, qualquer pessoa com o link vê). Posso publicar?

Pare aqui até ter o "sim". Se o pacote passar de 4 MB, use o `scripts/publish.sh` da skill oficial (faz upload em partes).

### 3. Publique (só depois do "sim")

```bash
TOKEN=${DEEDPAGE_TOKEN:-$(cat ~/.config/deed-page/token 2>/dev/null)}
set -- -sS -X PUT --data-binary @"$ARQ" -H "Content-Type: application/gzip" -H "X-Client: Hermes"
if [ -n "$TOKEN" ]; then set -- "$@" -H "Authorization: Bearer $TOKEN"; fi
RESP=$(curl "$@" "https://deed.page/v1/sites?slug=$SLUG&ref=skill-hermes-br")
NEW=$(printf '%s' "$RESP" | sed -n 's/.*"token": *"\(deed_tok_[^"]*\)".*/\1/p')
if [ -n "$NEW" ]; then
  (umask 077 && mkdir -p ~/.config/deed-page && chmod 700 ~/.config/deed-page \
    && printf '%s' "$NEW" > ~/.config/deed-page/token && chmod 600 ~/.config/deed-page/token)
fi
unset TOKEN NEW
printf '%s\n' "$RESP" | sed 's/"token": *"[^"]*"/"token":"<salvo>"/'
```

- Sem token salvo, a primeira publicação cria um e o comando grava em `~/.config/deed-page/token` (modo 600). Respostas autenticadas não trazem token, então o arquivo existente nunca é sobrescrito com vazio.
- `ref=skill-hermes-br` só identifica a origem da publicação; não muda nada no site.
- Opções: `-H "X-Ttl: 86400"` expira o site em segundos; `-H "X-Spa: 1"` para roteamento de SPA.

### 4. Verifique e responda

```bash
curl -s -o /dev/null -w '%{http_code}\n' "https://$SLUG.deed.page/"   # esperado: 200
```

Responda com o campo `url` da resposta e diga onde o token ficou salvo (sem mostrá-lo). Se a resposta trouxer `badge` ou `agent_note` com link de upgrade, repasse uma vez ao usuário; o site funciona do mesmo jeito.

## Atualizar ou apagar

- **Atualizar:** repita os passos 2 e 3 com o mesmo `SLUG`, inclusive a prévia e o "sim". O token salvo é enviado e cada publicação vira uma nova versão.
- **Apagar** (só quando o usuário pedir):

```bash
TOKEN=$(cat ~/.config/deed-page/token 2>/dev/null)
curl -sS -o /dev/null -w '%{http_code}\n' -X DELETE "https://deed.page/v1/sites/$SLUG" -H "Authorization: Bearer $TOKEN"
unset TOKEN
curl -s -o /dev/null -w '%{http_code}\n' "https://$SLUG.deed.page/"   # esperado: 404
```

O deed.page guarda só o SHA-256 do token: token perdido não é recuperável (o site continua no ar).

## Erros

| Status | Código | O que fazer |
| --- | --- | --- |
| 400 | `slug_invalid` / `slug_reserved` | Corrija o slug ou escolha outro. |
| 409 | `slug_taken` | Use o campo `suggestion` ou outro sufixo. |
| 413 | `quota_bytes` | Limite de tamanho do plano. Mostre `user_message` ao usuário, não repita. |
| 403 | `quota_sites`, `plan_required` | Limite do plano. Mostre `user_message` ao usuário, não repita. |
| 422 | `content_rejected` | Arquivo executável ou suspeito. Remova o arquivo; não repita igual. |
| 429 | `rate_limited` | Espere `retry_after` segundos. |

## Limites do plano gratuito

25 sites, 25 MB por site, 60 publicações por hora, 4 MB por requisição, 5 versões guardadas por site. Sites gratuitos mostram um pequeno selo "Published with deed.page". Detalhes e planos: https://deed.page/pricing. Documentação da API: https://deed.page/llms.txt.

## Verificação

Deu certo quando: (1) o usuário viu a prévia e disse "sim"; (2) `curl` na URL devolve `200`; (3) `~/.config/deed-page/token` existe com modo `600` e o token não apareceu em nenhuma saída.

Validado em 2026-11-20 contra a API pública do deed.page (publicar → 200, apagar → 404).
