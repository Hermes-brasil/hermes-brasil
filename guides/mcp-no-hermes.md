# Servidores MCP no Hermes — guia prático

MCP (Model Context Protocol) é o padrão que conecta o Hermes a **ferramentas externas**: CRMs, bancos de dados, GitHub, navegador, planilhas, e-mail. Em vez de escrever integração na mão, você conecta um "servidor MCP" e as ferramentas dele passam a aparecer para o agente.

Este guia mostra o caminho oficial por linha de comando (o mesmo que o Hermes usa por dentro), do catálogo pronto até os pitfalls que quebram na prática.

Confira a sua versão antes de começar: `hermes --version`.

## 1. O caminho mais rápido: o catálogo

O Hermes traz um catálogo de servidores aprovados, com instalação de um comando:

```bash
hermes mcp catalog              # lista os servidores disponíveis (e quais você já tem)
hermes mcp install <nome>       # instala um do catálogo (ex: hermes mcp install deepwiki)
hermes mcp install official/<nome>
```

Depois de instalar, confira como ficou:

```bash
hermes mcp list
```

A saída mostra **nome, transporte, ferramentas e status** de cada servidor:

```text
Name             Transport                      Tools        Status
github           npx -y ...                     all          ✓ enabled
playwright       npx -y @playwright/mcp         all          ✓ enabled
```

## 2. Conectar um servidor que não está no catálogo

### Servidor local (stdio, geralmente via `npx`)

```bash
hermes mcp add meu-servidor --command npx --args -y @escopo/pacote-mcp
```

> `--args` precisa ser o **último** parâmetro do comando — tudo que vier depois dele é tratado como argumento do servidor.

### Servidor remoto (HTTP/SSE)

```bash
hermes mcp add meu-servidor --url https://mcp.exemplo.com/mcp
hermes mcp add meu-servidor --url https://mcp.exemplo.com/mcp --auth oauth
hermes mcp add meu-servidor --url https://mcp.exemplo.com/mcp --auth header
```

### Passando credenciais para o servidor

```bash
hermes mcp add meu-servidor --command npx --args -y @escopo/pacote-mcp --env API_KEY=valor MINHA_VAR=outro
```

**Só as variáveis declaradas no `--env` são repassadas ao processo do servidor** — o resto do seu ambiente é removido. Não conte com variáveis globais do shell dentro do servidor.

Para servidores com OAuth, use `--auth oauth` e, quando o token expirar:

```bash
hermes mcp login <nome>          # força nova autenticação
hermes mcp reauth <nome>         # reautentica um
hermes mcp reauth --all          # reautentica todos
```

## 3. Testar antes de confiar

```bash
hermes mcp test <nome>
```

O comando conecta de verdade e lista as ferramentas descobertas:

```text
  Testing 'meu-servidor'...
  Transport: HTTP → https://mcp.exemplo.com/mcp
  Auth: none
  ✓ Connected (3578ms)
  ✓ Tools discovered: 5
    nome_da_ferramenta       Descrição do que ela faz...
```

Isso é a sua prova de que o servidor responde e de **quais ferramentas** ele expõe. Se um servidor expõe 40 ferramentas e você só usa 3, dá para reduzir:

```bash
hermes mcp configure <nome>      # escolhe quais ferramentas ficam habilitadas
hermes mcp remove <nome>         # remove o servidor
```

## 4. Usar as ferramentas na sessão

Servidores MCP são carregados **no início da sessão**. Se você acabou de adicionar um servidor e as ferramentas não aparecem, reinicie a sessão ou o gateway:

```bash
hermes gateway restart
```

> **Pitfall:** rodar `hermes gateway restart` **de dentro** do próprio processo do gateway pode ser bloqueado. Rode o comando de um shell externo (SSH/console do VPS).

## 5. Expor o Hermes como servidor MCP

O caminho inverso também existe: dar as ferramentas do Hermes para outro agente.

```bash
hermes mcp serve
```

## 6. Pitfalls que quebram na prática

1. **A saída de `hermes mcp test` pode vazar a sua chave.** O comando imprime o transporte completo — em servidores configurados com chave na query string, a URL aparece **com a chave dentro**. Nunca cole essa saída em issue, PR, print ou log público. Se vazou, rotacione a chave.
2. **Editar `~/.hermes/config.yaml` na mão é arriscado.** Reescrever o arquivo com um dumper YAML genérico pode reindentar chaves de primeiro nível para dentro de um bloco de servidor e quebrar tudo (`hermes mcp list` passa a dar erro). Prefira os comandos `hermes mcp ...`; se editar, faça backup antes e valide com `hermes mcp list` depois.
3. **Pacote `npx` que não existe.** Se o servidor não sobe, confirme o nome publicado: `npm view @escopo/pacote-mcp version`.
4. **Servidor remoto sem autenticação é superfície de ataque.** Prefira `--auth oauth`/`--auth header` e endpoints oficiais do fornecedor; desconfie de URL de terceiro que pede sua chave.
5. **Servidor local roda com os seus privilégios.** Um servidor stdio é **código que você executa na sua máquina** — com acesso ao seu usuário. Instale só o que você confia e leia o que ele faz. Se precisar de isolamento, rode o Hermes (ou o servidor) em container/usuário dedicado.
6. **Timeout de conexão.** Servidor que demora para subir pode estourar o tempo de descoberta:

```bash
hermes mcp add meu-servidor --command npx --args -y @escopo/pacote-mcp --connect-timeout 60
```

## 7. Quando NÃO usar MCP

- Se a ferramenta tem uma **CLI boa**, chamar a CLI pelo terminal costuma ser mais simples e mais fácil de depurar do que instalar um servidor MCP.
- Se você usa a integração **uma vez por mês**, não vale manter o servidor habilitado (ele carrega em toda sessão). Instale quando precisar, remova depois.
- MCP não é atalho para "dar superpoderes" ao agente sem revisar: cada servidor novo é uma credencial e um processo a mais rodando no seu ambiente.

## Recursos

- [Documentação oficial do Hermes](https://hermes-agent.nousresearch.com/docs)
- [Repositório do Hermes Agent](https://github.com/NousResearch/hermes-agent)
- [Especificação do MCP](https://modelcontextprotocol.io)

## Dúvidas

Abra uma issue neste repositório ou traga no grupo da comunidade Hermes Brasil (links no README).
