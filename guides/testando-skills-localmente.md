# Testando skills antes do Pull Request

Uma skill precisa funcionar no Hermes, não apenas parecer correta no Markdown. Este roteiro testa uma contribuição publicada na branch do seu fork antes de pedir revisão.

## 1. Revisão estática

Confirme que o arquivo está em `skills/<nome>/SKILL.md` e começa com frontmatter válido:

```yaml
---
name: nome-da-skill
description: Descreva o que faz e quando o Hermes deve usá-la.
---
```

Antes do teste:

```bash
git diff --check
git status --short
```

Revise também:

- comandos, caminhos e URLs citados;
- ausência de senhas, tokens e dados pessoais;
- seção de verificação com resultado observável;
- instruções de erro para ações que podem perder dados.

## 2. Publique a branch no seu fork

```bash
git push -u origin sua-branch
```

Monte a URL raw do `SKILL.md`:

```text
https://raw.githubusercontent.com/SEU-USUARIO/hermes-brasil/SUA-BRANCH/skills/NOME/SKILL.md
```

## 3. Inspecione antes de instalar

```bash
hermes skills inspect URL-RAW
```

Confira nome, descrição, origem e alertas da análise de segurança. Não use `--force` para esconder um alerta sem entender a causa.

## 4. Instale com nome temporário

O nome temporário evita substituir uma skill já instalada:

```bash
hermes skills install URL-RAW --name NOME-teste
hermes skills list | grep NOME-teste
```

Skills instaladas entram em novas sessões. Inicie uma sessão nova para o teste.

## 5. Teste o comportamento

Teste um pedido que deve ativar a skill:

```bash
hermes chat --toolsets skills -q "Use a skill NOME-teste para TAREFA-REAL"
```

Depois teste:

1. um pedido fora do escopo, para conferir se a skill não invade outras tarefas;
2. uma entrada incompleta ou inválida;
3. o principal risco descrito na própria skill;
4. a verificação final indicada no `SKILL.md`.

Uma resposta convincente não basta. Confira o arquivo, comando ou estado que a skill prometeu produzir.

## 6. Remova a instalação de teste

```bash
hermes skills uninstall NOME-teste
```

Registre no Pull Request os comandos executados e o resultado. Se alguma parte não pôde ser testada, diga qual e por quê.

## Referência oficial

- [Creating Skills](https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills)
- [Working with Skills](https://hermes-agent.nousresearch.com/docs/guides/work-with-skills)
