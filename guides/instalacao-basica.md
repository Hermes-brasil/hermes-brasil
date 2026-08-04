# Guia de Exemplo: Instalando o Hermes Agent

Guia de exemplo em português para a comunidade. Este guia mostra o padrão de documentação para a pasta `guides/`.

## Pré-requisitos

- Python 3.10+
- Um VPS ou máquina com acesso ao terminal

## Instalação

```bash
# Instalar via pipx (recomendado)
pipx install hermes-agent

# Ou clonar e instalar
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
pip install -e .
```

## Primeira execução

```bash
# Verificar instalação
hermes --version

# Iniciar o agente
hermes
```

## Configuração de modelo

Configure o provedor de modelo (ex: DeepSeek, OpenAI, Anthropic):

```bash
hermes config set model.provider deepseek
hermes config set model.name deepseek-chat
```

## Configurar Telegram

```bash
hermes gateway setup
# Siga as instruções para conectar o Telegram
```

## Recursos

- [Docs oficial](https://hermes-agent.nousresearch.com)
- [Repositório](https://github.com/NousResearch/hermes-agent)

## Dúvidas

Abra uma issue ou fale no grupo da comunidade Hermes Brasil.
