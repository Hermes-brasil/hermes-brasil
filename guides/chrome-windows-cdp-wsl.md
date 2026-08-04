# Chrome do Windows no Hermes/WSL via CDP

Este guia conecta o Hermes Agent executado no WSL2 a uma instância visível e persistente do Google Chrome no Windows usando o Chrome DevTools Protocol (CDP).

É útil quando o agente precisa:

- trabalhar numa janela que você consegue acompanhar;
- reutilizar cookies e sessões autenticadas de um perfil dedicado;
- operar sites dinâmicos com `browser_navigate`, `browser_snapshot`, `browser_click` e outras ferramentas do Hermes;
- evitar o custo de um browser em nuvem em tarefas que podem rodar localmente.

> [!CAUTION]
> CDP concede controle amplo sobre o navegador: leitura do DOM, navegação, cliques e acesso às sessões abertas naquele perfil. Use um perfil exclusivo, mantenha a porta apenas em loopback e nunca exponha o endpoint na internet ou na rede local.

## Arquitetura

```text
Hermes Agent no WSL2
        │
        │ HTTP/WebSocket CDP em 127.0.0.1:18802
        ▼
Google Chrome no Windows
        │
        └── perfil dedicado e persistente
```

A documentação oficial do Hermes recomenda `chrome-devtools-mcp` quando o Chrome do Windows não está alcançável diretamente pelo WSL2. O acesso CDP direto deste guia é uma alternativa prática quando o loopback do Windows está disponível dentro do WSL, por exemplo com `networkingMode=mirrored`.

## Pré-requisitos

- Windows com WSL2;
- Google Chrome instalado no Windows;
- Hermes Agent instalado no WSL;
- interoperabilidade WSL → PowerShell funcionando;
- toolset `browser` habilitado no Hermes.

Confira o Hermes:

```bash
hermes --version
hermes tools list
```

## 1. Inicie um Chrome dedicado com CDP

Abra o **Windows PowerShell** e execute:

```powershell
$Port = 18802
$Profile = Join-Path $env:USERPROFILE ".hermes-chrome-windows"
$StartUrl = "https://example.com"

$Chrome = @(
  (Join-Path $env:ProgramFiles "Google\Chrome\Application\chrome.exe"),
  (Join-Path ${env:ProgramFiles(x86)} "Google\Chrome\Application\chrome.exe")
) | Where-Object { $_ -and (Test-Path $_) } | Select-Object -First 1

if (-not $Chrome) {
  throw "Google Chrome não encontrado. Ajuste a variável `$Chrome para o caminho instalado."
}

$ProfileArg = "--user-data-dir=`"$Profile`""

Start-Process -FilePath $Chrome -ArgumentList @(
  "--remote-debugging-address=127.0.0.1",
  "--remote-debugging-port=$Port",
  $ProfileArg,
  "--no-first-run",
  "--no-default-browser-check",
  "--new-window",
  $StartUrl
)
```

O diretório `.hermes-chrome-windows` é separado do perfil pessoal padrão. Faça login manualmente nos sites necessários dentro dessa janela; os cookies persistirão nas próximas execuções do mesmo perfil.

### Por que `--user-data-dir` é obrigatório?

Desde o Chrome 136, as flags de depuração remota não são respeitadas no diretório de dados padrão do Chrome. Além disso, uma instância já aberta pode absorver a nova janela e ignorar `--remote-debugging-port`. Um diretório dedicado força um processo separado e reduz o risco de expor o perfil pessoal.

Se o Chrome abrir, mas a porta não responder, feche somente as janelas desse perfil dedicado e tente novamente. Antes de encerrar qualquer processo, salve o trabalho aberto. Para identificar processos que usam o perfil:

```powershell
$Profile = Join-Path $env:USERPROFILE ".hermes-chrome-windows"
Get-CimInstance Win32_Process -Filter "Name='chrome.exe'" |
  Where-Object { $_.CommandLine -like "*$Profile*" } |
  Select-Object ProcessId, CommandLine
```

Evite `Stop-Process -Name chrome -Force`: isso também fecha as janelas pessoais do usuário.

## 2. Verifique o CDP no Windows

Ainda no PowerShell:

```powershell
$Version = Invoke-RestMethod "http://127.0.0.1:18802/json/version" -TimeoutSec 5
$Version | Select-Object Browser, webSocketDebuggerUrl
```

Resultado esperado:

```text
Browser                 webSocketDebuggerUrl
-------                 ---------------------
Chrome/<versão>         ws://127.0.0.1:18802/devtools/browser/<id>
```

Também é possível listar as páginas abertas:

```powershell
Invoke-RestMethod "http://127.0.0.1:18802/json/list" -TimeoutSec 5 |
  Where-Object { $_.type -eq "page" } |
  Select-Object title, url, id
```

## 3. Verifique o endpoint a partir do WSL

No terminal WSL:

```bash
curl -fsS --max-time 5 http://127.0.0.1:18802/json/version
```

Se retornar JSON com `Browser` e `webSocketDebuggerUrl`, o caminho direto está pronto.

### Windows responde, mas o WSL não

Primeiro confirme a diferença:

- `Invoke-RestMethod` funciona no Windows;
- `curl` falha no WSL.

Nesse cenário, prefira uma destas opções:

1. **Rota oficial recomendada:** conecte o `chrome-devtools-mcp` do lado Windows ao Hermes pelo suporte MCP.
2. **Rede espelhada do WSL2 (Windows 11 22H2+):** habilite o compartilhamento de loopback em `%UserProfile%\.wslconfig`:

```ini
[wsl2]
networkingMode=mirrored
autoProxy=true
```

Depois, salve todo o trabalho no WSL e execute no Windows PowerShell:

```powershell
wsl --shutdown
```

Abra o WSL novamente e repita o `curl`.

> [!WARNING]
> Não resolva o problema vinculando o CDP em `0.0.0.0`, criando redirecionamento público de porta ou liberando a porta no firewall. Quem alcança o endpoint pode controlar o perfil do navegador.

## 4. Configure o Hermes

Com o endpoint já saudável:

```bash
hermes tools enable browser
hermes config set browser.cdp_url http://127.0.0.1:18802
```

Inicie uma nova sessão do Hermes para que as ferramentas condicionadas ao CDP sejam registradas:

```bash
hermes
```

Para uma conexão temporária no CLI interativo, também é possível usar o WebSocket retornado por `/json/version`:

```text
/browser connect ws://127.0.0.1:18802/devtools/browser/<id>
/browser status
```

`/browser connect` é um comando do CLI interativo. Em Telegram, Discord, WebUI e outros gateways ele não é processado como slash command; nesses casos, prefira `browser.cdp_url` no `config.yaml`.

## 5. Faça um smoke test seguro

Peça ao Hermes:

```text
Abra https://example.com no browser e informe somente o título e a URL.
Não clique em botões nem execute ações públicas.
```

Fluxo esperado:

1. `browser_navigate` abre a URL no Chrome visível;
2. `browser_snapshot` lê a árvore de acessibilidade;
3. o agente retorna o título e a URL;
4. nenhuma ação externa é executada.

Para inspecionar manualmente os alvos expostos pelo Chrome:

```bash
curl -fsS --max-time 5 http://127.0.0.1:18802/json/list
```

## Diagnóstico rápido

| Sintoma | Causa provável | Correção segura |
|---|---|---|
| Chrome abre, mas `/json/version` falha no Windows | A nova janela foi absorvida por outra instância ou o perfil dedicado já estava aberto sem CDP | Feche somente o perfil dedicado, relance e verifique a porta |
| Windows responde, WSL dá timeout | Loopback do Windows não está compartilhado com o WSL | Use `chrome-devtools-mcp` ou `networkingMode=mirrored` |
| Hermes abre um browser vazio/deslogado | Foi usado outro `user-data-dir` | Confirme o caminho do perfil dedicado e faça login manualmente nele |
| `browser_cdp` reclama que a URL não é WebSocket | Um comando CDP bruto recebeu a URL HTTP de descoberta | Use o `webSocketDebuggerUrl` completo retornado por `/json/version` |
| A página existe, mas um target antigo trava | Aba/processo ficou degradado | Abra uma aba nova e repita o smoke test antes de reiniciar tudo |
| A ferramenta CDP não aparece | O endpoint estava indisponível ao iniciar a sessão ou o toolset está desabilitado | Verifique a porta, habilite `browser` e inicie uma nova sessão |

## Regras de segurança operacional

- Use um perfil dedicado para automação; não reutilize o perfil pessoal padrão.
- Mantenha `--remote-debugging-address=127.0.0.1`.
- Não exponha a porta por túnel público, roteador, firewall ou `0.0.0.0`.
- Não copie cookies, tokens ou arquivos do perfil para repositórios.
- Separe perfis por cliente ou contexto sensível.
- Trate postar, enviar mensagens, realizar pagamentos, publicar, excluir e alterar produção como ações que exigem aprovação humana explícita.
- Prefira `browser_snapshot` para leitura e refs atuais para cliques; não reutilize seletores de uma página antiga sem verificar o estado novamente.
- Para tarefas apenas informativas, use leitura primeiro e pare antes de qualquer mutação externa.

## Referências oficiais

- [Hermes Agent — Browser Automation](https://hermes-agent.nousresearch.com/docs/user-guide/features/browser)
- [Hermes Agent — Configuration: Browser](https://hermes-agent.nousresearch.com/docs/user-guide/configuration#browser)
- [Hermes Agent — MCP no WSL2 com Chrome do Windows](https://hermes-agent.nousresearch.com/docs/guides/use-mcp-with-hermes#wsl2-bridge-hermes-in-wsl-to-windows-chrome)
- [Chrome for Developers — Changes to remote debugging switches](https://developer.chrome.com/blog/remote-debugging-port)
- [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/)
