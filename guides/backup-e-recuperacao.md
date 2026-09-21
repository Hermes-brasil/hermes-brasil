# Backup e recuperação do Hermes

Backup só conta quando o arquivo pode ser verificado e restaurado. O Hermes oferece backup completo para recuperação ou migração e snapshot rápido para mudanças de rotina.

## Backup completo ou rápido

Use backup completo para recuperar toda a instalação ou migrar de máquina:

```bash
hermes backup
```

O arquivo inclui configuração, perfis, sessões, skills, cron jobs e credenciais. O código do `hermes-agent` e caches regeneráveis ficam fora.

Use snapshot rápido antes de uma atualização ou mudança pequena:

```bash
hermes backup --quick --label "antes-da-mudanca"
```

O snapshot rápido guarda apenas estado crítico. Ele não substitui o backup completo para migração.

## Verifique o arquivo

O comando informa o caminho final do `.zip`. Substitua o nome abaixo pelo arquivo exibido, teste o conteúdo e gere um checksum no mesmo diretório:

```bash
HERMES_BACKUP_FILE=hermes-backup-AAAA-MM-DD.zip
python3 -m zipfile -t "${HERMES_BACKUP_FILE:?}"
sha256sum "${HERMES_BACKUP_FILE:?}" > "${HERMES_BACKUP_FILE:?}.sha256"
chmod 600 "${HERMES_BACKUP_FILE:?}" "${HERMES_BACKUP_FILE:?}.sha256"
```

Depois de copiar os dois arquivos para outro disco ou máquina:

```bash
sha256sum -c hermes-backup-AAAA-MM-DD.zip.sha256
```

Guarde pelo menos uma cópia fora da máquina do Hermes.

## Proteja o backup

O backup completo contém `.env`, `auth.json` e outros segredos. Não envie o arquivo para repositório, issue, chat, link público ou armazenamento sem proteção adequada. Trate o `.zip` como uma cópia das credenciais da instalação.

## Restauração

**A restauração sobrescreve arquivos existentes no diretório do Hermes.** Pare os gateways e crie um ponto de retorno da instalação de destino antes de importar. Não use `--force` na primeira tentativa.

```bash
hermes gateway stop --all
hermes backup -o ~/hermes-antes-da-restauracao.zip
hermes import ~/hermes-backup-AAAA-MM-DD.zip
```

Em uma máquina nova, instale primeiro o Hermes Agent e copie o backup para ela. O import restaura os dados, não o código do programa.

## Verificação depois do import

```bash
hermes doctor
hermes status --deep
hermes skills list
hermes cron list
```

Confirme também:

- provedor e autenticação com `hermes setup`;
- profiles esperados;
- próxima execução dos cron jobs;
- canais de mensagem antes de iniciar os gateways.

Quando a verificação terminar:

```bash
hermes gateway start --all
```

Se a restauração falhar, preserve mensagens de erro e o backup original. Não repita com `--force` sem entender qual arquivo entrou em conflito.

## Teste de recuperação

Periodicamente, restaure uma cópia em máquina ou ambiente descartável. Verifique skills, sessões, cron jobs e um fluxo real sem apontar canais para usuários. Testar no ambiente principal não prova recuperação e ainda pode substituir dados válidos.

## Referência oficial

- [Comandos `hermes backup` e `hermes import`](https://hermes-agent.nousresearch.com/docs/reference/cli-commands#hermes-backup)
- [Exportando o Hermes para outra máquina](https://hermes-agent.nousresearch.com/docs/reference/faq#exporting-hermes-to-another-machine)
