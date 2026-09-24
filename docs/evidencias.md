# Evidências e limites de reprodução

[← Visão geral](../README.md)

## Base desta publicação

O write-up foi preparado a partir do registro de implementação, incluindo saídas de comandos, campos de eventos e confirmações feitas durante os testes. Os resultados abaixo são históricos. Nenhuma VM foi acessada para repetir os testes durante a redação desta versão.

| Marco | Evidência registrada | O que permite concluir |
|---|---|---|
| Agente Windows | Mensagem de conexão em `ossec.log` e agente no dashboard | O endpoint conectou-se ao Manager |
| Sysmon | Event ID 1 local e canal reconhecido pelo coletor | A fonte gerou eventos e foi configurada para coleta |
| Regra `100100` | ID, nível, descrição e campos de processo no alerta | O caso Notepad/PowerShell foi detectado |
| Regra `100101` | Evento com ID final `100101` após ajuste de `if_sid` | O caso testado passou pela regra filha |
| Oracle | Conexão ao Manager via Tailscale registrada pelo agente | Houve comunicação remota funcional |
| auditd | `decoder.name=auditd`, chave de teste, arquivo, comando e regra `80781` | A escrita controlada foi auditada e gerou alerta |
| Graylog | Contadores de Input aumentando e evento no stream | Alertas foram recebidos e indexados |
| Parser | Resultado de `wazuh_rule_id:100100` em mensagem nova | O campo extraído ficou pesquisável |

## O que não está sendo afirmado

- Não houve teste de carga, benchmark ou medição de disponibilidade.
- Não foi demonstrada ausência de perdas no encaminhamento UDP.
- Não foi calculada taxa de falsos positivos ou cobertura de técnicas MITRE.
- Não há exportação integral das configurações nem backup das VMs neste repositório.
- As capturas originais não foram incluídas nesta edição; não foram substituídas por imagens simuladas.

## Próxima coleta de evidências

Uma rodada de reprodução pode acrescentar capturas sanitizadas e amostras exportadas, com data, versão, ação executada e consulta usada. A sequência recomendada é:

1. Estado dos dois agentes e serviços relevantes.
2. Evento local Sysmon com processo pai e filho.
3. Alertas `100100` e `100101` com suas descrições e severidades.
4. Evento auditd com chave, arquivo e identidades de usuário.
5. Input, stream e configuração da pipeline Graylog.
6. Evento novo com os seis campos `wazuh_*` e consulta correspondente.

Antes de publicar novas evidências, remover credenciais, chaves, identificadores pessoais de conexão e dados não necessários à análise. Preservar IDs de regras, nomes genéricos dos endpoints e campos técnicos suficientes para sustentar a conclusão.
