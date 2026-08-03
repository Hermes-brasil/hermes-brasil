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
├── guides/          → Guias de instalação, configuração e uso (pt-BR)
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

- **Grupo WhatsApp**: (adicionar link quando disponível)
- **Hermes Atlas**: [hermesatlas.com](https://hermesatlas.com) — mapa do ecossistema
- **Docs oficial**: [hermes-agent.nousresearch.com](https://hermes-agent.nousresearch.com)

## 📜 Licença

[MIT](LICENSE) © 2026 Comunidade Hermes Brasil
