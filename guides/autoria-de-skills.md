# Autorando skills que o agente realmente carrega

Uma skill só existe quando o agente **a carrega na hora certa**. Skill bonita que nunca é acionada é documentação caríssima: ocupa espaço, dá manutenção e não muda nenhum comportamento.

Este guia é sobre o único problema que importa na autoria de skills: **fazer o agente abrir a sua skill exatamente na tarefa para a qual ela foi escrita** — e não abrir nas outras.

## 1. Como o Hermes decide carregar uma skill

Entenda o mecanismo antes de escrever uma linha. O Hermes usa *progressive disclosure* em dois níveis:

1. **Sempre no contexto**: de toda skill instalada, o agente vê apenas duas coisas — o `name` e a `description`. É isso que aparece na lista de skills disponíveis em cada sessão.
2. **Sob demanda**: o corpo do `SKILL.md` (e os arquivos ligados) só entra no contexto quando o agente chama a ferramenta de leitura da skill, decidindo que aquela skill é relevante.

Consequência direta e desconfortável: **o corpo da sua skill pode ser perfeito e ainda assim nunca ser lido.** Quem decide se a skill existe na prática é a `description`. Ela é o gatilho, não o resumo.

Consequência secundária, igualmente importante: **o corpo só deve conter o que não caberia na `description`.** Se a skill é grande, ela é grande porque contém comandos exatos, tabelas de decisão, pitfalls e templates — não porque explica o óbvio.

## 2. O gatilho mora nos primeiros 57 caracteres

O Hermes trunca e ranqueia descrições em listas compactas. Na prática, o padrão que funciona é:

```yaml
description: Use quando <gatilho concreto>. <o que a skill faz em uma linha>.
```

Os primeiros caracteres precisam ser **autoexplicativos e concretos**, porque é isso que o agente lê primeiro ao decidir. O resto da frase é comportamento.

Ruim (não aciona nada):

```yaml
description: Uma coleção abrangente de boas práticas, dicas e insights sobre como trabalhar de forma eficiente com automações modernas.
```

Bom (aciona na tarefa certa):

```yaml
description: Use quando o usuário pedir para agendar tarefa recorrente no Hermes. Cria cron com ledger, delivery e HOME correto.
```

Teste rápido: **leia só a sua `description` e pergunte "que tarefa faz um agente abrir isso?"** Se a resposta for "qualquer coisa relacionada a produtividade", você escreveu um texto de marketing, não um gatilho.

### Gatilho, não tema

| Tema (ruim) | Gatilho (bom) |
|---|---|
| `Ferramentas de imagem` | `Use ao gerar thumbnail para vídeo do YouTube` |
| `Integração com o Google` | `Use ao ler ou escrever planilha do Google Sheets` |
| `Notícias e atualidades` | `Use ao montar o resumo diário de notícias de IA` |
| `Boas práticas de código` | `Use ao revisar código gerado por IA antes de aplicar no site` |

A diferença é o **verbo e o objeto**: "gerar", "ler", "revisar" + o artefato concreto.

## 3. Formato mínimo e o que a CI valida

```markdown
---
name: nome-da-skill
description: Use quando <gatilho>. <comportamento em uma linha>.
---

# Nome da skill

<instruções>
```

Regras verificadas automaticamente pelo workflow do repositório (`.github/workflows/validar-skills.yml`):

- a primeira linha do `SKILL.md` é `---` e o frontmatter fecha com outro `---`;
- `name` **igual ao nome da pasta** (`skills/nome-da-skill/SKILL.md` → `name: nome-da-skill`);
- `description` presente e com no máximo **1024 caracteres** (limite do padrão agentskills.io). Acima de 200, o workflow emite aviso não bloqueante — descrição longa é sintoma de gatilho mal definido;
- varredura por padrões de segredo (tokens GitHub, `sk-`, AWS, Slack, chaves privadas) em todo o repositório.

Rode a validação antes de abrir o PR. Ela é a mesma que roda no CI.

## 4. Corpo: escreva para um agente, não para um humano

Você não tem um leitor humano paciente navegando em seções. Você tem um agente que precisa **executar**. Isso muda o texto:

- **Comandos exatos, não direções.** `gh api repos/X/Y --jq '.stars'` aciona; "consulte a API do GitHub" não aciona nada.
- **Regra + o porquê.** Uma linha de justificativa evita que a próxima sessão "otimize" a regra e quebre o resultado. Regra sem motivo vira sugestão.
- **Pitfall com sintoma observável.** Não escreva "cuidado com o timeout". Escreva: "se a conexão cair em 60s, o proxy derruba antes do build terminar — use `--max-time 300`".
- **Ordem de execução, não catálogo.** Um procedimento numerado que o agente segue vale mais que cinco seções de conceito.
- **Verificação explícita ao final.** Todo caminho precisa terminar em "como saber que deu certo" — comando, status HTTP, contagem esperada.

Evite:

- listas de "boas práticas" genéricas que o modelo já domina;
- adjetivos sem lastro ("robusto", "poderoso", "completo");
- seções "Introdução"/"Conclusão" que não instruem nada;
- `etc.` — o agente não infere o que você não escreveu.

## 5. Disclosório progressivo: SKILL.md enxuto, arquivos ligados

O corpo carrega inteiro quando a skill é acionada. Se a sua skill tem 800 linhas de referência, todo esse texto entra no contexto toda vez — mesmo quando o agente só precisava do comando principal.

Divida assim:

```
skills/nome-da-skill/
├── SKILL.md              → gatilho + procedimento principal (o que sempre importa)
├── references/           → tabelas, catálogos, material de consulta por tópico
├── templates/            → arquivos base para copiar
└── scripts/              → automação executável
```

O agente enxerga a lista dos arquivos ligados junto com a skill e carrega só o que a tarefa pedir. Regra prática: **se um bloco do corpo não é usado em toda execução, ele pertence a `references/`.**

Cuidado com arquivos ligados que ninguém cita. Um `references/api.md` que não é mencionado no `SKILL.md` fica invisível na prática — aponte o caminho no corpo: "para o catálogo completo de endpoints, veja `references/api.md`".

## 6. Skills que envelhecem

- **Referência obsoleta é pior que referência ausente.** Se um arquivo ligado é removido, o ponteiro no corpo tem de sair no mesmo commit; um marcador órfão faz o agente achar que a informação existe.
- **Versione o que muda.** Se a skill depende de um comando, modelo ou endpoint que evolui, carimbe a versão validada no texto ("validado na v0.20.6") para que o próximo leitor saiba quando desconfiar.
- **Corrigiu no uso? Corrija na skill.** Ao descobrir um passo que faltava ou um comando que falhou, atualize a skill na mesma sessão. Skill que não aprende com o uso é cópia estática.

## 7. Quando **não** criar uma skill

- **O modelo já sabe fazer.** "Como escrever um loop em Python" não precisa de skill.
- **É fato, não procedimento.** Quem é o cliente, qual o IP do servidor, qual a convenção de tom: isso é memória, não skill. Memória entra em todo contexto; skill entra sob demanda.
- **É progresso temporário.** "Terminei a etapa 3" não pertence a lugar nenhum permanente.
- **É preferência sobre um tipo de tarefa.** Aí sim é skill — preferências e correções do usuário para *aquela* tarefa pertencem à skill que cobre a tarefa.

O critério: **procedimento reutilizável com gatilho reconhecível → skill. Contexto que vale para toda sessão → memória.**

## 8. Como saber que a skill carrega de verdade

Não confie na leitura. Teste o acionamento:

1. **Instale** a skill (veja `guides/testando-skills-localmente.md` para o roteiro completo com branch de fork):

   ```bash
   hermes skills inspect URL-RAW
   hermes skills install URL-RAW --name nome-teste
   hermes skills list | grep nome-teste
   ```

2. **Confirme que ela aparece na lista** com o nome e a descrição que você escreveu. É exatamente isso que o agente vê.

3. **Dê a tarefa-gatilho em uma sessão nova.** Abra uma sessão limpa e peça a tarefa que a sua `description` promete — sem citar o nome da skill. Ex.: se o gatilho é "revisar código gerado por IA", peça "revise esse trecho que a IA gerou antes de subir".

4. **Verifique o acionamento.** O agente deve carregar a skill por conta própria. Se ele resolveu a tarefa *sem* abrir a skill, o problema está na `description` (gatilho vago ou tema em vez de verbo) — reescreva e repita.

5. **Teste o caminho negativo.** Peça uma tarefa vizinha, mas não coberta pela skill. Se ela for carregada à toa, a `description` está larga demais.

O step 5 é o que quase ninguém faz — e é o que separa uma skill precisa de uma skill que rouba contexto de todas as outras tarefas.

## 9. Checklist antes do PR

- [ ] Arquivo em `skills/<nome>/SKILL.md`, pasta com o mesmo nome do campo `name`
- [ ] `description` começa com gatilho concreto e **testado no passo 8.3**
- [ ] Teste negativo (8.5) feito: a skill não é carregada em tarefa vizinha
- [ ] Comandos citados foram executados por você, não copiados de memória
- [ ] Nada de segredo, caminho privado ou dado pessoal (a CI varre, mas a revisão é sua)
- [ ] Seção de verificação com resultado observável
- [ ] Blocos de consulta movidos para `references/` e citados no corpo
- [ ] `hermes skills list` mostra nome e descrição corretos

## Referências

- Padrão de formato de skill: [agentskills.io](https://agentskills.io)
- Teste local antes do PR: [`guides/testando-skills-localmente.md`](./testando-skills-localmente.md)
- Fluxo de contribuição: [`CONTRIBUTING.md`](../CONTRIBUTING.md)
