# Casos de detecção e validação

[← Visão geral](../README.md)

Os três casos abaixo foram validados durante o laboratório. Os comandos reproduzem ações benignas. Execute-os apenas em endpoints de teste, com as fontes de telemetria e regras previamente configuradas. As consultas pressupõem um intervalo de tempo que inclua a nova execução.

## Caso 1 — Notepad iniciado pelo PowerShell

| Campo | Definição |
|---|---|
| Objetivo | Validar a cadeia de criação de processo até o alerta |
| Endpoint | `WIN11-01` |
| Fonte | Sysmon, Event ID 1 |
| Regra | `100100`, nível 5 |
| Condição | Imagem termina em `notepad.exe`; processo pai termina em `powershell.exe` |
| Mapeamento usado | MITRE ATT&CK `T1059.001` — PowerShell |

No PowerShell do endpoint:

```powershell
Start-Process notepad.exe
```

No Wazuh:

```text
agent.name:"WIN11-01" AND rule.id:100100
```

Depois da integração e parsing, no stream `Wazuh Alerts` do Graylog:

```text
wazuh_rule_id:100100
```

**Resultado registrado:** alerta `100100`, nível 5, descrição `LAB TEST: Notepad launched from PowerShell.`. O teste também foi repetido com sucesso na segunda VM Windows antes de sua retirada.

**Triagem:** comparar imagem, processo pai, linhas de comando, usuário, hashes e horário. O Notepad pode aparecer sob `WindowsApps` em instalações modernas; uma regra baseada apenas em um caminho fixo poderia perder esse caso.

**Limite:** é uma regra de validação do laboratório. Abrir Notepad a partir de PowerShell é uma ação legítima; o mapeamento MITRE descreve o mecanismo observado e não prova comprometimento. A comparação pelo nome final do executável também não verifica sua autenticidade.

## Caso 2 — PowerShell com `-EncodedCommand`

| Campo | Definição |
|---|---|
| Objetivo | Identificar o comportamento coberto pela regra nativa e aplicar uma regra filha |
| Fonte | Sysmon, Event ID 1 |
| Regra final | `100101`, nível 8 |
| Regra pai | `92057` no ruleset observado |
| Mapeamento usado | MITRE ATT&CK `T1059.001` |

No PowerShell da `WIN11-01`:

```powershell
$command = 'Write-Output "WAZUH_TEST"'
$encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($command))
powershell.exe -EncodedCommand $encoded
```

O conteúdo codificado apenas imprime `WAZUH_TEST`. No Wazuh:

```text
agent.name:"WIN11-01" AND rule.id:100101
```

**Resultado registrado:** `100101`, nível 8, descrição `PowerShell executed with an encoded command.`. Antes do ajuste, o mesmo comportamento havia sido classificado pela regra nativa `92057`, nível 12.

**Correção aplicada:** a regra personalizada passou a depender de `<if_sid>92057</if_sid>`. A ausência de `100101` não significava ausência de detecção; era necessário consultar o evento e verificar a regra final efetivamente selecionada.

**Limites:** essa versão depende do comportamento e da existência de `92057` no ruleset instalado. Não é uma regra genérica para qualquer processo ou variante de PowerShell. O nível 8 preserva o exercício realizado; não é uma recomendação de reduzir a severidade nativa em produção. Administração legítima também pode utilizar comandos codificados.

## Caso 3 — Escrita auditada no Linux

| Campo | Definição |
|---|---|
| Endpoint | `ORACLE-UBUNTU-01` |
| Fonte | `/var/log/audit/audit.log` |
| Arquivo monitorado | `/var/tmp/wazuh_audit_test` |
| Chave | `wazuh_audit_test` |
| Regra observada | `80781`, nível 3 |

Na Oracle, com auditd ativo e coleta configurada:

```bash
sudo touch /var/tmp/wazuh_audit_test
sudo auditctl -w /var/tmp/wazuh_audit_test -p wa -k wazuh_audit_test
sudo auditctl -l
```

No Manager, a lista `/var/ossec/etc/lists/audit-keys` deve conter uma única entrada correspondente:

```text
wazuh_audit_test:write
```

O cadastro da chave no Manager faz parte do teste. A lista deve estar carregada na configuração e a alteração deve ser aplicada conforme a versão instalada. Consulte a [configuração oficial de auditoria](https://documentation.wazuh.com/current/user-manual/capabilities/system-calls-monitoring/audit-configuration.html).

Gere uma escrita na Oracle:

```bash
echo "SOC_LAB_TEST $(date -Is)" | sudo tee -a /var/tmp/wazuh_audit_test > /dev/null
```

Consulte no Wazuh:

```text
agent.name:"ORACLE-UBUNTU-01" AND data.audit.key:"wazuh_audit_test"
```

**Resultado registrado:** decoder `auditd`, comando `tee`, `success=yes`, arquivo esperado e regra `80781`. O usuário original era `ubuntu`, enquanto UID/EUID indicavam execução com root via sudo. O retorno `exit=3`, nesse evento bem-sucedido de abertura de arquivo, representava um descritor, não um código de erro.

**Limite:** o watch foi criado em tempo de execução. Não foi demonstrada sua persistência após reinicialização. A chave também não transforma todas as alterações Linux em eventos de alta relevância; o caso demonstra a auditoria do recurso escolhido.

## Como avaliar uma repetição dos testes

1. Confirmar serviço e fonte local antes de consultar o SIEM.
2. Registrar horário, endpoint e ação executada.
3. Consultar inicialmente por endpoint e janela temporal; depois filtrar por regra.
4. Conferir o conteúdo do evento, não apenas a contagem de resultados.
5. Para o Graylog, verificar recebimento, stream e criação dos campos separadamente.
6. Registrar também um teste negativo, como executar a ação fora da condição da regra, antes de ampliar o uso da detecção.

O último item é uma melhoria metodológica proposta. Não há uma bateria de testes negativos documentada como concluída no histórico.
