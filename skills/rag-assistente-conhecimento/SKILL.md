---
name: rag-assistente-conhecimento
description: Criar assistente RAG com conhecimento da empresa.
---

# RAG — Assistente de Conhecimento (Retrieval-Augmented Generation)

> **O que é:** técnica que permite IA acessar conhecimento EXTERNO (documentos,
> FAQ, banco de dados da empresa) para dar respostas precisas e baseadas nos
> dados do cliente — em vez de respostas genéricas do modelo.

## Conceitos-chave

```
📋 PIPELINE DE RAG:
├── 1. INGESTÃO: carregar documentos
├── 2. INDEXAÇÃO: dividir em chunks + gerar embeddings
├── 3. RETRIEVAL: buscar chunks relevantes à pergunta
└── 4. GERAÇÃO: modelo responde usando o contexto recuperado
```

### Técnicas avançadas (aplicar conforme necessidade)
```
├── Query Translation: melhorar a pergunta antes de buscar
│   ├── Multi-Query (várias versões da pergunta)
│   ├── RAG Fusion (combina resultados)
│   ├── Decomposition (quebra pergunta complexa)
│   ├── Step Back (pergunta mais geral)
│   └── HyDE (gera resposta hipotética para buscar)
├── Routing: direcionar pergunta ao modelo/fluxo certo
├── Indexing avançado: RAPTOR, ColBERT
├── CRAG (Conditional RAG): corrigir retrieval com base no grau de confiança
└── Adaptive RAG: decide quando buscar mais dados ou responder direto
```

## Embeddings
- Convertem texto em vetores numéricos (capturam significado)
- Similaridade de COSSENO (padrão), Euclidiana, Dot Product
- Modelos: OpenAI, Gemini, Cohere, open source (sentence-transformers)
- **pt-BR:** `paraphrase-multilingual-MiniLM-L12-v2` (entende português)
- **CRÍTICO:** usar o MESMO modelo para documentos e perguntas

```python
from sentence_transformers import SentenceTransformer
modelo = SentenceTransformer("paraphrase-multilingual-MiniLM-L12-v2")
doc = modelo.encode("Horário de funcionamento da clínica")
pergunta = modelo.encode("A que horas abre?")
```

## Chunking — lição importante
NÃO dividir por contagem de palavras (mistura assuntos e recupera seção errada).
Dividir por **SEÇÕES temáticas** (títulos `##`), cada seção vira um chunk com
o título prefixado como contexto: `f"{titulo}: {corpo}"`. Isso mantém cada
chunk coerente e a busca fica muito mais precisa.

```python
def chunking_por_secoes(texto):
    """Divide texto em seções (##) em vez de cortar por palavras."""
    secoes = []
    atual = {"titulo": "geral", "corpo": []}
    for linha in texto.splitlines():
        if linha.startswith("## "):
            if atual["corpo"]:
                secoes.append(atual)
            atual = {"titulo": linha[3:].strip(), "corpo": []}
        elif linha.strip():
            atual["corpo"].append(linha.strip())
    if atual["corpo"]:
        secoes.append(atual)
    return [f"{s['titulo']}: {' '.join(s['corpo'])}" for s in secoes if s["corpo"]]
```

## Busca semântica (retrieval)
```python
import numpy as np

def buscar(pergunta, chunk_embeddings, textos, top_k=3):
    q = modelo.encode(pergunta)
    scores = [np.dot(q, c) / (np.linalg.norm(q) * np.linalg.norm(c)) for c in chunk_embeddings]
    melhores = np.argsort(scores)[-top_k:][::-1]
    return [(textos[i], scores[i]) for i in melhores]
```

## Fluxo de trabalho (para cliente real)

```
⚙️ IMPLEMENTAÇÃO:
├── 1. Levantar os documentos do cliente (FAQ, políticas, catálogo)
├── 2. Estruturar o conteúdo (seções claras)
├── 3. Rodar chunking + embeddings
├── 4. Testar busca com perguntas reais
├── 5. Conectar geração (API do modelo)
├── 6. Integrar ao canal (WhatsApp/site)
├── 7. Testar respostas e ajustar
├── 8. Definir limite (quando passar para humano)
└── 9. Manutenção e atualização do conhecimento
```

## Venda como serviço (preços de referência)
```
💰 PRECIFICAÇÃO:
├── Chatbot básico: R$ 1.000-3.000
├── Assistente RAG completo: R$ 3.000-8.000
├── Assistente avançado (multi-função): R$ 8.000-15.000
├── Manutenção mensal: R$ 500-2.000
└── Licença/uso recorrente: R$ 300-1.500/mês
```

## Pitfalls
- Chunking por seção, não por palavra (crítico para qualidade)
- Embeddings locais = busca sem custo; usar modelo multilingue para pt-BR
- Testar sempre com perguntas reais do cliente
- Definir limite claro de quando passar para atendimento humano
- Proteger dados sensíveis (LGPD) — não vazar informações confidenciais
- Atualizar o conhecimento periodicamente
- Transparência: avisar que é assistente de IA

## Quando usar
- Cliente pede chatbot/assistente inteligente
- Criar produto/ebook de RAG (tema com demanda)
- Quer diferenciar de chatbot genérico
