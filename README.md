# 🇧🇷 Hermes Brasil

**Comunidade brasileira de Hermes Agent** — o agente de IA open source da Nous Research que cresce com você.

Este repositório reúne skills, guias, integrações e recursos em **português** para a comunidade brasileira de Hermes Agent.

## O que é o Hermes Agent?

[Hermes Agent](https://github.com/NousResearch/hermes-agent) é um agente de IA open source (MIT) da Nous Research, com um loop de aprendizado próprio: ele cria skills a partir da experiência, melhora durante o uso e constrói um modelo profundo de quem você é. Roda num VPS barato, GPU cluster ou serverless.

- 📝 **Skills**: conhecimento sob demanda, compatível com o padrão [agentskills.io](https://agentskills.io)
- 🧠 **Memória**: persistente, sobrevive entre sessões
- 🔌 **MCP**: integra servidores MCP para mais ferramentas
- ⏰ **Cron**: automações agendadas
- 💬 **Canais**: Telegram, Discord, Slack, WhatsApp, Signal, Teams

## 📁 Estrutura do repositório

```
hermes-brasil/
├── skills/          → Skills em português (formato SKILL.md)
│   ├── rag-assistente-conhecimento/    → Assistente RAG com conhecimento da empresa
│   ├── prospeccao-b2b-comercio-local/  → Prospecção B2B de comércio local
│   ├── hermes-philosophy-podcast/      → Filosofia do Hermes (podcast Nous Research)
│   └── orquestracao-kanban-humano-agente/ → Kanban com humanos + agentes de IA
├── guides/          → Guias de instalação, configuração e uso (pt-BR)
│   └── instalacao-basica.md
├── integrations/    → Integrações (WhatsApp, n8n, etc.)
├── community/       → Recursos da comunidade brasileira
├── README.md        → Este arquivo
└── LICENSE          → MIT
```

## 🚀 Como contribuir

1. Faça um **fork** deste repositório
2. Crie uma branch para sua contribuição
3. Adicione sua skill/guia/integração (veja os modelos abaixo)
4. Abra um **Pull Request**

### Padrão de skill (SKILL.md)

Cada skill em `skills/` segue o formato [agentskills.io](https://agentskills.io):

```markdown
---
name: nome-da-skill
description: O que a skill faz (primeira frase = gatilho)
---
# Conteúdo da skill em português
```

## 🤝 Comunidade

Junte-se aos canais oficiais da comunidade brasileira de Hermes Agent:

- **Grupo WhatsApp**: [Entre no grupo](https://chat.whatsapp.com/DIDJlL9yQdlBY0fa4pABbe?s=cl&p=a&ilr=1)
- **Telegram**: [t.me/hermesagentbr](https://t.me/hermesagentbr)
- **Discord**: [Entre no Discord](https://discord.gg/nkZQeVjcJ)
- **Hermes Bible**: [www.hermesbible.com](https://www.hermesbible.com/) — guia e recursos da comunidade
- **Docs oficial**: [hermes-agent.nousresearch.com](https://hermes-agent.nousresearch.com)

## 📜 Licença

[MIT](LICENSE) © 2026 Comunidade Hermes Brasil
