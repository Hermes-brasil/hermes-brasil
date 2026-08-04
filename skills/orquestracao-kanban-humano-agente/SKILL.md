---
name: orquestracao-kanban-humano-agente
description: Orquestrar projetos com Kanban humano + agentes de IA.
---

# Orquestração Kanban — Humanos + Agentes de IA

> Padrão validado pelo co-fundador da Nous Research (Hermes Agent) no podcast com Peter Yang (02/08/2026). Usado para orquestrar trabalho onde humanos e agentes de IA colaboram no MESMO board.

## Princípio central

**UM só Kanban** para humanos E agentes de IA. Não são dois sistemas separados — são as mesmas colunas, mesmas tarefas, mas as posições são **intercambiáveis** (humano ou IA em qualquer ponto).

```
"Posso ter um gerente de projeto humano no board manualmente
enquanto agentes preenchem as peças que ele pediu."
— Karan Malhotra (Nous Research)
```

## Colunas padrão do board

```
📊 6 COLUNAS:
├── Triage       → ideias cruas entram
├── To-do        → tarefa aguardando (dependência)
├── Ready        → tarefa designada, pronta pra pegar
├── In Progress  → executor (humano ou IA) trabalhando
├── Blocked      → TRAVOU, precisa de humano/intervenção
└── Done         → concluído
```

## Mecânica de orquestração

```
⚙️ ORQUESTRADOR/DISPATCHER:
├── Quebra meta grande em tarefas menores
├── Atribui a executores (humanos ou agentes especialistas)
├── Checa o board periodicamente (ex: a cada 60s)
├── Roda tarefas na ordem certa
└── Detecta "Blocked" e escalona pra humano

💾 DURABILIDADE (crítico):
├── Board persistente (ex: SQLite local)
├── Cada handoff/tarefa fica SALVO
├── Cada executor é um processo com identidade própria
└── Trabalho NÃO se perde entre sessões
```

## Orquestração em qualquer direção

```
🔄 QUEM INSTRUI QUEM (intercambiável):
├── Humano (gerente) → instrui agentes
│   Ex: "crie as 5 tasks, agente executa"
├── Agente → instrui humanos
│   Ex: agente orienta equipe de call center
└── A mesma tarefa pode ser feita por humano OU IA
```

## Exemplos de uso

```
✅ EXEMPLOS PRÁTICOS:
├── Criação de conteúdo longo (ebook, curso):
│   ├── Agente escreve capítulo por capítulo
│   ├── Humano (gerente) dirige o ritmo e revisa
│   └── Mesmo board, papéis trocáveis
├── Prospecção/operações multi-etapa
├── Projetos que precisam de sequência + revisão humana
└── Qualquer fluxo com ponto de decisão humana

💡 COMO APLICAR:
├── Usar coluna "Blocked" explicitamente
│   (agente trava → marca e pede decisão humana)
├── Quebrar metas grandes em tasks no board
└── Manter progresso durável entre sessões
```

## Quando usar esta skill
- Projetos que precisam de sequência de tarefas + revisão humana
- Orquestrar agentes + humanos no mesmo fluxo
- Trabalho longo que não deve se perder entre sessões
- Automação multi-etapa com ponto de decisão humana

## Boas práticas
- Definir colunas claras (Triage → Done)
- Marcar explicitamente quando tarefa fica "Blocked"
- Salvar progresso duravelmente (não confiar só em memória de sessão)
- Deixar claro em cada tarefa se é humano ou IA que executa
- Usar o board como fonte de verdade do progresso
