---
name: think-tank-research
description: Use quando precisar de pesquisa multi-persona com síntese protegida por evidências.
version: 0.1.0
author: Mateus Fardin
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [pesquisa, multiagente, personas, sintese, relatorios, think-tank]
    related_skills: [grounded-citations]
---

# Skill de Pesquisa Think Tank

Use esta skill para conduzir um processo de pesquisa multi-persona: vários subagentes especializados investigam o mesmo tema por lentes distintas e, depois, uma camada coordenadora de síntese preserva divergências, compara evidências e produz um relatório consolidado. Isto não é um exercício de interpretação de papéis; as personas são lentes metodológicas com mandatos, padrões de evidência e contratos de saída explícitos.

## Quando usar

Use quando o usuário pedir:

- um relatório com múltiplas perspectivas, lentes especialistas ou revisão simulada por comitê;
- pesquisa estratégica para decisões de negócio, comunicação, produto, mercado, tecnologia, política pública ou editorial;
- um “think tank”, “banca”, “comitê”, “painel de especialistas” ou “personas pesquisadoras”;
- visões contrastantes antes de uma recomendação;
- um relatório consolidado que diferencie consenso, divergência, incerteza e ação.

Não use para:

- consulta factual simples ou respostas baseadas em uma única fonte;
- tarefas em que o usuário já especificou um método único e autoritativo;
- decisões jurídicas, médicas, financeiras ou de segurança de alto risco sem ressalvas explícitas e verificação em fontes primárias;
- gerar citações falsas, citações fabricadas ou consenso artificial.

## Pré-requisitos

- Se o tema exigir fatos atuais ou externos, use `web_search` / `web_extract` ou ferramentas de navegador; não dependa apenas da memória.
- Se o usuário forneceu arquivos, URLs, pastas ou sessões anteriores, inspecione essas fontes antes de delegar.
- Se a tarefa for ampla, defina o brief de pesquisa antes de criar subagentes.
- Use `delegate_task` para trabalho paralelo de personas com escopo delimitado. Para pesquisas duráveis ou de longa duração, use `cronjob`, um processo `terminal` em segundo plano ou um subprocesso do Hermes.

## Conceitos centrais

### Persona

Uma persona é um papel de pesquisa definido por:

1. **Mandato** — aquilo que esta persona é responsável por enxergar.
2. **Lente** — como ela avalia evidências e trade-offs.
3. **Contrato de saída** — as seções exatas que deve retornar.
4. **Limites** — o que ela não deve fazer.

Evite personas baseadas apenas em tom, estilo ou identidade ficcional. Prefira papéis funcionais, como estrategista de mercado, analista técnico, analista de risco, revisor cético, consultor de implementação, analista de políticas públicas, pesquisador cultural ou estrategista editorial.

### Think tank

O think tank é a camada de síntese. Ele não apenas calcula uma média de opiniões. Ele deve:

- identificar convergências e divergências;
- comparar a qualidade das evidências;
- sinalizar pressupostos frágeis;
- preservar visões minoritárias importantes;
- produzir uma posição consolidada justificada;
- tornar a incerteza visível.

## Procedimento

### 1. Estruture o brief de pesquisa

Estabeleça, a partir do usuário ou do contexto disponível:

```markdown
## Tema
## Pergunta central
## Objetivo do report
## Público-alvo
## Escopo geográfico / temporal
## Profundidade desejada
## Tipos de fonte aceitáveis
## Restrições e riscos
## Número de personas
## Formato final desejado
```

Se um campo ausente mudar materialmente o trabalho, faça uma única pergunta de esclarecimento. Caso contrário, declare premissas razoáveis e prossiga.

Critério de conclusão: a pergunta de pesquisa, o público, o escopo e o formato de saída estão explícitos.

### 2. Selecione personas deliberadamente

Escolha de 3 a 7 personas, salvo se o usuário especificar outra quantidade. Para a maioria dos relatórios estratégicos, use cinco por padrão:

1. **Estrategista de mercado** — demanda, posicionamento, modelos de negócio, concorrência e incentivos econômicos.
2. **Analista técnico** — viabilidade, ferramentas, arquitetura, restrições e requisitos de implementação.
3. **Especialista de domínio/comunicação** — público, linguagem, marca, adoção e implicações culturais.
4. **Analista de risco / advogado do diabo** — modos de falha, compliance, fragilidade operacional e promessas infladas.
5. **Consultor de implementação** — roadmap, sequenciamento, recursos, ganhos rápidos e governança.

Adapte o elenco ao domínio. Para temas acadêmicos, inclua um revisor de literatura. Para temas de políticas públicas, inclua lentes regulatórias e de impacto social. Para temas de marca/conteúdo, inclua lentes editoriais e de audiência.

Critério de conclusão: toda persona tem um mandato distinto e não sobreposto.

### 3. Dê a cada persona um prompt independente

Cada subagente deve receber o mesmo brief central, além de um mandato específico da persona. Inclua este contrato de saída:

```markdown
# Parecer da Persona: [nome]

## Mandato aplicado
## Tese principal
## Achados relevantes
## Evidências e fontes
## Oportunidades
## Riscos / objeções
## Pontos que outras lentes tendem a negligenciar
## Recomendações
## Grau de confiança
Alto / Médio / Baixo, com justificativa
## Perguntas em aberto
```

Limites a incluir em todo prompt de persona:

- Não faça a síntese do comitê inteiro.
- Não busque consenso.
- Não suavize as objeções do seu papel.
- Diferencie evidência, inferência e especulação.
- Cite fontes ou diga explicitamente quando uma afirmação se baseia em raciocínio a partir do brief.
- Retorne apenas o parecer da sua persona.

Critério de conclusão: cada subagente recebe um prompt autossuficiente e consegue trabalhar sem ler a conversa principal.

### 4. Rode a pesquisa das personas em paralelo

Use `delegate_task(tasks=[...])` quando as tarefas forem independentes. Passe contexto suficiente para cada agente-filho: o brief, o mandato da persona, o formato esperado, o idioma, os requisitos de citação e as restrições.

Para tarefas com uso intenso de web, escolha uma das abordagens:

- permitir que cada subagente faça sua própria pesquisa, se as ferramentas estiverem disponíveis e o escopo for pequeno; ou
- coletar primeiro um pacote de fontes compartilhado com `web_search` / `web_extract` e depois passá-lo aos subagentes para reduzir navegação duplicada.

Critério de conclusão: toda persona planejada retorna um parecer, ou qualquer parecer ausente é explicitamente marcado como indisponível.

### 5. Pontue as evidências antes da síntese

Antes de escrever o relatório final, construa um mapa de evidências:

```markdown
## Fonte / evidência
## Personas que usaram
## Tipo: primária / secundária / opinião / inferência
## Força: alta / média / baixa
## Observações de confiabilidade
```

Trate afirmações repetidas sem fonte como fracas, mesmo que várias personas as repitam. Dê mais peso a fontes primárias, documentação oficial, dados transparentes e evidências recentes quando a atualidade for relevante.

Critério de conclusão: a síntese diferencia evidência forte de afirmações fracas repetidas.

### 6. Construa a matriz de convergência

Crie uma matriz como:

| Tema | Persona A | Persona B | Persona C | Persona D | Síntese |
|---|---|---|---|---|---|
| [questão] | posição | posição | posição | posição | consenso/divergência |

Inclua pelo menos:

- consensos;
- divergências reais;
- pressupostos;
- lacunas de evidência;
- implicações para a decisão do usuário.

Critério de conclusão: nenhuma divergência importante fica escondida ou achatada.

### 7. Produza o relatório final

Estrutura padrão:

```markdown
# Report Think Tank: [tema]

## 1. Sumário executivo
## 2. Pergunta investigada e escopo
## 3. Personas participantes
## 4. Principais consensos
## 5. Principais divergências
## 6. Evidências mais fortes
## 7. Hipóteses frágeis e lacunas
## 8. Cenários ou leituras possíveis
## 9. Recomendação consolidada
## 10. Plano de ação / próximos passos
## 11. Pareceres individuais resumidos
## 12. Apêndice de fontes
```

Para relatórios longos, salve o relatório completo em um arquivo com `write_file` e entregue o caminho/link na resposta final. Para respostas no chat, inclua o sumário executivo e as matrizes mais importantes.

Critério de conclusão: o relatório inclui tanto as lentes individuais quanto uma visão consolidada justificada.

## Guardrails

### Evite falsa diversidade

Ruim: cinco personas com nomes diferentes, mas o mesmo mandato.

Bom: cada persona tem um critério de decisão diferente e é instruída a enfatizar o que outras podem deixar passar.

Verificação: se duas personas provavelmente citariam as mesmas evidências e chegariam ao mesmo tipo de conclusão, una ou redefina uma delas.

### Evite consenso artificial

A síntese não deve transformar discordância em compromisso genérico. Quando uma objeção minoritária for importante, preserve-a assim:

```markdown
## Divergência preservada
Embora a maioria das personas conclua X, a persona Y discorda por causa de Z. A síntese adota X apenas sob as condições A/B/C.
```

### Evite superficialidade multiplicada

Cinco pesquisas rasas não equivalem a profundidade. Exija que cada persona responda a uma subpergunta específica do seu papel e identifique a qualidade das fontes.

### Separe fatos de interpretações

Rotule afirmações como:

- **Evidência** — diretamente sustentada por uma fonte ou pelo material fornecido.
- **Inferência** — conclusão raciocinada a partir de evidências.
- **Hipótese** — plausível, mas não confirmada.
- **Opinião estratégica** — recomendação ou juízo decisório.

### Não fabrique citações

Se uma fonte não foi inspecionada, não a cite como evidência. Se o acesso à pesquisa falhar, diga isso e reduza o grau de confiança.

### Preserve o contexto do usuário com segurança

Quando o relatório envolver negócios privados, material de clientes ou documentos confidenciais, passe apenas o contexto necessário aos subagentes. Não exponha segredos, dados pessoais, credenciais ou material privado irrelevante.

### Use confiança com honestidade

Toda persona e a síntese final devem trazer um grau de confiança com justificativa. Baixa confiança é aceitável quando as fontes são escassas, fatos atuais estão inacessíveis ou a pergunta é especulativa.

## Template de Prompt da Persona

```markdown
Você é a persona: [NOME].

## Brief comum
[Tema, pergunta central, objetivo, público, escopo, formato final]

## Seu mandato
[Responsabilidade específica]

## Sua lente metodológica
[Critérios, prioridades, tipos de evidência que mais importam]

## Limites
- Não faça a síntese geral.
- Não busque consenso.
- Não suavize objeções importantes do seu papel.
- Separe evidência, inferência, hipótese e opinião estratégica.
- Não invente fontes, dados ou citações.

## Saída obrigatória
# Parecer da Persona: [nome]

## Mandato aplicado
## Tese principal
## Achados relevantes
## Evidências e fontes
## Oportunidades
## Riscos / objeções
## Pontos que outras lentes tendem a negligenciar
## Recomendações
## Grau de confiança
## Perguntas em aberto
```

## Template de Prompt de Síntese

```markdown
Você é o coordenador de um think tank multiagente.

## Tarefa
Consolidar os pareceres individuais abaixo sem apagar divergências relevantes.

## Regras
- Não trate repetição como evidência forte por si só.
- Preserve divergências importantes.
- Diferencie evidência, inferência, hipótese e opinião estratégica.
- Identifique lacunas e pressupostos frágeis.
- Gere uma recomendação consolidada apenas quando justificável.

## Entregáveis
1. Matriz de convergência.
2. Consensos.
3. Divergências.
4. Evidências fortes.
5. Hipóteses frágeis.
6. Recomendação consolidada.
7. Plano de ação.
8. Apêndice com resumo de cada persona.

## Pareceres
[Inserir pareceres individuais]
```

## Verificação

Antes de finalizar, verifique:

- [ ] O brief está explícito o suficiente para orientar a pesquisa.
- [ ] As personas têm mandatos distintos e não viram analistas genéricos.
- [ ] Cada persona retornou as seções obrigatórias.
- [ ] A qualidade das evidências foi avaliada, não apenas contada.
- [ ] As divergências foram preservadas e explicadas.
- [ ] A recomendação final decorre das evidências e reconhece incertezas.
- [ ] As fontes são citadas apenas quando foram de fato inspecionadas.
- [ ] Contexto privado ou sensível não foi exposto desnecessariamente a subagentes.
