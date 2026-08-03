---
name: prospeccao-b2b-comercio-local
description: Prospeccao B2B de comercio local (Google Places + WhatsApp + registro).
---

# Prospecção B2B — Comércio Local

> Pipeline automatizado de prospecção B2B para comércio local.
> Cobre coleta → qualificação → envio → registro → follow-up.
> Nichos: clínicas, salões, pet shops, restaurantes, mercados, lojas, oficinas, franquias.

## Visão geral do fluxo

```
1. COLETA: Google Places API → nomes, endereços, telefones, sites, avaliações
2. DEDUP: consultar banco ANTES de inserir (zero duplicatas)
3. FILTRO: manter só comércios com WhatsApp OU Instagram OU site
4. ENVIO: WhatsApp (Evolution API) com delay entre mensagens
5. REGISTRO: SQLite + planilha (Google Sheets)
6. RESUMO: relatório no Telegram
```

## 1. Coleta — Google Places API (New) — RECOMENDADO

Muito mais rápido e confiável que browser:

```bash
curl -s "https://places.googleapis.com/v1/places:searchText?key={API_KEY}" \
  -H "Content-Type: application/json" \
  -H "X-Goog-FieldMask: places.displayName,places.formattedAddress,places.nationalPhoneNumber,places.rating,places.websiteUri" \
  -d '{"textQuery":"auto pecas em Pirituba Sao Paulo","maxResultCount":10}'
```

**⚠️ Header OBRIGATÓRIO:** `X-Goog-FieldMask` — sem ele retorna erro 400.

### Setup (uma única vez)
1. `console.cloud.google.com` → ativar **Places API**
2. **ATIVAR FATURAMENTO** (obrigatório mesmo com crédito grátis)
3. US$ 200/mês de crédito cobre ~20.000 buscas
4. Gerar chave em Credentials > API Key

| Erro | Causa | Solução |
|---|---|---|
| `REQUEST_DENIED` + "enable Billing" | Faturamento não propagou | Aguardar 5-30 min |
| `PERMISSION_DENIED` | Chave restrita | Liberar no console |
| 400 "FieldMask required" | API Nova sem header | Adicionar `X-Goog-FieldMask` |

## 2. Dedup — Consultar banco ANTES de adicionar

```python
import sqlite3, re
db = sqlite3.connect("leads.db")
cur = db.cursor()
digits = re.sub(r'\D', '', raw_phone)
cur.execute("SELECT id FROM leads WHERE telefone = ?", (digits,))
if cur.fetchone() is None:
    cur.execute("INSERT INTO leads ... VALUES (...)")
else:
    print(f"⬇️ Duplicata ignorada: {nome}")
```

## 3. Filtro — WhatsApp OU Instagram OU Site

Regra prática: incluir comércios com WhatsApp **OU** Instagram **OU** site.

```python
import re
digits = re.sub(r'\D', '', phone)
# Número móvel brasileiro = 11 dígitos, DDD + '9'
if (len(digits) == 11 and digits[2] == '9') or instagram or website:
    manter = True
else:
    manter = False  # fixo sem Instagram/site -> descartar
```

## 4. Envio via WhatsApp (Evolution API)

```python
import urllib.parse, requests
instance = urllib.parse.quote("nome-instancia")
r = requests.post(
    f"{EVOLUTION_URL}/message/sendText/{instance}",
    json={"number": "5511XXXXXXXXX", "text": mensagem,
          "options": {"delay": 120000}},  # 2 min entre mensagens
    headers={"apikey": API_KEY, "Content-Type": "application/json"})
```

**⚠️ Delay de 2 MINUTOS entre mensagens** (evita bloqueio do WhatsApp).
Use apenas com número que aceite contato não solicitado dentro das regras da plataforma.

## 5. Registro no SQLite

Após cada envio:
- Sucesso → `status='enviado'`
- Número inexistente → `status='fixo'`
- Erro → `status='erro_envio'`

## 6. Abordagem de porta (presencial — alto diferencial)

Dono de pequeno negócio confia mais em quem aparece. **Não vender — mostrar.**

```
"Oi, tudo bem? Sou da região, trabalho com um sistema que atende
seus clientes pelo WhatsApp 24h. Tem 2 minutos? vou te mostrar."

[Mostra o sistema no celular do dono]

"Olha: o cliente pergunta o preço, o sistema responde na hora.
Ele pergunta o estoque, o sistema consulta e responde. Automático."

"Quer testar 15 dias grátis? Se não gostar, não paga nada."
```

**Indicação:** oferecer desconto — "Seu amigo fechar, você ganha 1 mês grátis."

## Pitfalls
- Google Maps (browser) cai sem login e com CAPTCHA → preferir Places API
- Nomes genéricos de loja retornam resultados de outras cidades → incluir bairro
- Places API NÃO retorna Instagram → buscar separadamente (web_search)
- Verificar conexão real do WhatsApp (profileName preenchido, não só status "open")
- Enviar com delay para não ser bloqueado
- Respeitar LGPD e boas práticas de prospecção

## Quando usar
- Prospecção B2B de comércio local
- Captar leads para automação WhatsApp
- Montar pipeline de vendas porta-a-porta + digital
