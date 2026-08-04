# Criando seu Primeiro Agente com Hermes

> Guia prático e básico para criar seu primeiro agente de IA com o Hermes Agent.
> Em português, passo a passo, do zero ao funcionando.

## O que é um agente no Hermes

No Hermes, um "agente" (perfil) é um **Hermes independente** com a própria:
- Configuração (config.yaml)
- Chaves de API (.env)
- Personalidade (SOUL.md)
- Memória, sessões, skills e estado

Cada agente é um "cérebro" separado para um propósito diferente — um assistente de código, um bot pessoal, um agente de pesquisa — sem misturar o estado de um com o outro.

## Pré-requisito

Ter o Hermes Agent instalado e funcionando. (Se ainda não instalou, veja o guia de [instalação](./instalacao-basica.md).)

## Passo 1 — Criar o perfil do agente

Abra o terminal e crie um perfil com o nome que quiser. Exemplo com `meuagente`:

```bash
hermes profile create meuagente
```

**Isso cria o agente E um comando próprio.** A partir de agora você tem:
- `meuagente setup` — para configurar
- `meuagente chat` — para conversar
- `meuagente gateway start` — para ativar em plataformas

> 💡 Dica: se o agente terá um papel específico (ex: pesquisa), defina uma descrição para o orquestrador saber o que ele faz bem:
> ```bash
> hermes profile create meuagente --description "Lê documentação e escreve resumos"
> ```

## Passo 2 — Configurar o agente

Configure as chaves de API, modelo e ferramentas do novo agente:

```bash
meuagente setup
```

Siga o assistente para:
1. Escolher o provedor (OpenAI, Anthropic, DeepSeek, etc.)
2. Escolher o modelo
3. Adicionar as chaves de API

> 💡 Dica rápida: para configurar tudo de uma vez com o Nous Portal (300+ modelos + ferramentas):
> ```bash
> meuagente setup --portal
> ```

## Passo 3 — Conversar com o agente

Agora é só conversar:

```bash
meuagente chat
```

O agente já tem skills básicas pré-instaladas. Teste pedindo algo simples:
```
"Resuma o que é o Hermes Agent"
```

## Passo 4 — Personalizar a personalidade (opcional)

Cada agente tem um arquivo `SOUL.md` que define sua personalidade. Edite para dar a ele a "cara" que quiser:

```bash
# Encontre o SOUL.md do agente
# Linux/macOS: ~/.hermes/profiles/meuagente/SOUL.md

# Estruture com 3 seções:
# 1. CORE TRUTHS — quem ele é (ex: "seja honesto, evite adulação")
# 2. BOUNDARIES — limites (ex: "respeite privacidade em grupos")
# 3. VIBES — tom (ex: "direto e claro")
```

## Passo 5 — Ativar em plataformas (opcional)

Para o agente responder no Telegram, WhatsApp, etc.:

```bash
meuagente gateway setup   # configura a plataforma
meuagente gateway start   # inicia o gateway
```

## Como clonar um agente existente

Em vez de começar do zero, você pode copiar a configuração de um agente já pronto:

```bash
# Copia só a config (mesmas chaves/modelo, memória nova)
hermes profile create trabalho --clone

# Copia tudo (config + skills + SOUL, memória nova)
hermes profile create backup --clone-all
```

Depois edite `~/.hermes/profiles/trabalho/SOUL.md` para dar outra personalidade.

## Resumo rápido

```
1. hermes profile create meuagente   → cria o agente
2. meuagente setup                   → configura modelo/chaves
3. meuagente chat                    → conversa
4. (opcional) editar SOUL.md         → personalidade
5. (opcional) gateway start          → ativa em plataformas
```

## Dicas

- **Um agente por propósito**: crie agentes separados para cada tarefa (código, pesquisa, bot pessoal) — não misture.
- **Descreva o papel**: use `--description` para o orquestrador saber o que o agente faz bem.
- **Clone em vez de refazer**: use `--clone` para reaproveitar config e skills.
- **Dê personalidade**: o `SOUL.md` é o que torna o agente único.

## Solução de problemas

| Problema | Solução |
|----------|---------|
| `comando não encontrado` | Recarregue o shell (`source ~/.bashrc`) |
| `API key not set` | Rode `meuagente setup` para configurar |
| Não consigo achar o SOUL.md | Verifique `~/.hermes/profiles/<nome>/SOUL.md` |

---

*Guia da comunidade Hermes Brasil — contribuição em português. Mais em [github.com/Hermes-brasil/hermes-brasil](https://github.com/Hermes-brasil/hermes-brasil).*
