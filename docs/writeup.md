# Implementação e decisões técnicas

[← Visão geral](../README.md)

## 1. Objetivo e método

O objetivo foi construir um caminho observável entre uma ação conhecida no endpoint e o alerta disponível para investigação. A implementação avançou por etapas: infraestrutura, primeiro agente, telemetria Windows, detecções, endpoint remoto e integração com Graylog.

Cada teste partiu de uma hipótese simples. Primeiro verificava-se se o sistema operacional registrava a ação; depois, se o agente coletava a fonte; por fim, se o evento gerava o alerta e aparecia na consulta esperada. Essa separação evitou interpretar ausência de resultado no dashboard como ausência de telemetria.

## 2. Infraestrutura local e rede

As VMs foram executadas no VirtualBox. Cada sistema local tinha conectividade NAT para instalação de pacotes e uma interface Host-Only para comunicação com os demais componentes e com o host físico.

| Sistema | Endereço na rede do laboratório |
|---|---|
| Host físico | `192.168.56.1` |
| Wazuh-SIEM | `192.168.56.10` |
| WIN11-01 | `192.168.56.21` |
| GRAYLOG-01 | `192.168.56.30` |

A interface Host-Only não recebeu gateway padrão; a saída para a Internet permaneceu na interface NAT. Essa topologia separou a comunicação local do laboratório da rede residencial, mas não foi tratada como isolamento absoluto: as VMs continuavam com saída para a Internet.

O limite prático foi a RAM do host. Durante a evolução, uma `WIN11-02` foi clonada, recebeu endereço e identidade próprios e passou pelos testes de detecção. Posteriormente, ela foi removida para dar lugar à infraestrutura Graylog. Essa decisão preservou a diversidade de fontes Windows/Linux sem manter um segundo Windows ligado.

## 3. Wazuh all-in-one

Manager, Indexer e Dashboard foram instalados no mesmo Ubuntu Server. O Filebeat completou o encaminhamento dos alertas ao Indexer. O acesso local ao dashboard foi feito por HTTPS no endereço do servidor.

A primeira instalação falhou durante a extração do Dashboard por falta de espaço. Embora existisse capacidade no disco virtual, o volume raiz LVM tinha aproximadamente 24 GB, enquanto outra parte do grupo de volumes permanecia livre. A ampliação do volume raiz permitiu concluir a instalação.

**Lição:** capacidade do VDI, tamanho do volume lógico e espaço livre no filesystem são medidas distintas. O diagnóstico precisou alcançar o filesystem que efetivamente recebia os pacotes.

Também ocorreram travamentos e avisos de RCU/soft lockup durante a virtualização. Houve ajustes de recursos e reinicializações, mas o registro não permite atribuir uma causa raiz única a todos esses episódios. A RAM do Wazuh chegou a ser aumentada e depois retornou a 8 GB.

## 4. Windows: agente e Sysmon

A `WIN11-01` foi cadastrada como agente `001`. Antes da instalação, foi validado o acesso ao Manager nas portas TCP 1514 e 1515. Após a inicialização do `WazuhSvc`, o log do agente confirmou a conexão.

O Sysmon ampliou a telemetria utilizada nos testes de criação de processos. O agente foi configurado para ler `Microsoft-Windows-Sysmon/Operational`, usando o formato `eventchannel`. O bloco correspondente está em [windows-sysmon.xml](../config/wazuh/windows-sysmon.xml).

Os testes utilizaram Event ID 1 para observar processo, processo pai, linha de comando, usuário e hashes. Esses campos permitem reconstruir uma execução; a presença deles, isoladamente, não caracteriza atividade maliciosa. A descrição dos eventos está na [documentação oficial do Sysmon](https://learn.microsoft.com/pt-br/sysinternals/downloads/sysmon).

### Primeira detecção: Notepad iniciado pelo PowerShell

A regra `100100` identificou a relação entre `powershell.exe` e `notepad.exe`. Seu propósito foi validar a cadeia completa com uma ação simples e controlada. A consulta por `rule.id:100100` retornou o alerta de nível 5.

### Segunda detecção: PowerShell com comando codificado

O teste de `-EncodedCommand` inicialmente apareceu sob a regra nativa `92057`, de nível 12. A regra personalizada `100101` foi então encadeada como filha dessa detecção. O novo teste retornou `100101`, de nível 8, conforme configurado no laboratório.

A dificuldade não foi causada pela indentação do XML. O ponto decisivo foi entender a hierarquia de regras e qual regra se tornava o alerta final. Em um ambiente operacional, reduzir a severidade de uma detecção nativa exigiria justificativa; o nível 8 aqui registra a escolha histórica do exercício. Consulte os [casos de detecção](deteccoes.md) para escopo e limitações.

## 5. Linux remoto por Tailscale

O servidor Ubuntu na Oracle Cloud foi cadastrado como `ORACLE-UBUNTU-01`, agente `002`. O Tailscale foi instalado no endpoint e no servidor Wazuh, permitindo comunicação entre eles sem publicar as portas do Manager na Internet.

O teste de conectividade mostrou uso de relay DERP em São Paulo, sem conexão direta estabelecida. A comunicação permaneceu funcional, e o log do agente confirmou conexão ao Manager por TCP 1514. Os endereços pessoais da tailnet foram omitidos desta publicação.

Após instalar auditd, foi necessário adicionar explicitamente a coleta de `/var/log/audit/audit.log` ao `ossec.conf` do agente. O serviço foi reiniciado, e o coletor registrou a abertura do arquivo.

Para o teste, foi criado um watch temporário sobre `/var/tmp/wazuh_audit_test`. A chave `wazuh_audit_test` também foi associada a `write` na lista CDB `audit-keys` do Manager. O evento passou a gerar a regra nativa `80781`, nível 3.

A evidência permitiu distinguir o usuário autenticado originalmente (`auid`) da identidade efetiva (`uid/euid`) usada com sudo. Esse vínculo foi relevante para atribuir a alteração ao usuário original, mesmo quando a escrita ocorreu com privilégios de root.

## 6. Graylog como destino adicional dos alertas

A `GRAYLOG-01` executou Graylog Open, Data Node e MongoDB com Docker Compose e volumes persistentes. O Data Node forneceu o backend de busca do Graylog; o Wazuh continuou utilizando seu próprio Indexer.

Durante a preparação, foi verificado `vm.max_map_count=1048576`. O valor já atendia ao requisito utilizado no laboratório; não foi necessário aumentá-lo. A configuração inicial incluiu o preflight, criação de CA e provisionamento dos certificados do Data Node.

O plano inicial previa Ubuntu 24.04, mas o histórico identificou Ubuntu 26.04 na VM efetivamente utilizada. A execução em containers funcionou no laboratório; esse resultado não estabelece uma matriz de suporte oficial. O diretório `/opt/graylog` também foi planejado, porém o diagnóstico posterior indicou que o Compose havia sido iniciado na home do usuário. Por isso, este repositório não apresenta um caminho não confirmado como configuração definitiva.

O Manager encaminhou alertas em JSON por `syslog_output` para `192.168.56.30:5140/UDP`. No Graylog, o Input final foi chamado `Wazuh Syslog`, com roteamento para `Wazuh Alerts`.

```text
Wazuh Manager
  → syslog_output / JSON / UDP 5140
  → Input: Wazuh Syslog
  → Default Routing
  → Stream: Wazuh Alerts
  → Pipeline: Wazuh Processing
  → Stage 0: Parse Wazuh JSON Final
```

O bloco de encaminhamento está em [graylog-forwarding.xml](../config/wazuh/graylog-forwarding.xml). A opção utilizada é descrita em [syslog_output — Wazuh](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/syslog-output.html).

## 7. Troubleshooting por camada

| Sintoma | Evidência que orientou o diagnóstico | Ajuste ou conclusão |
|---|---|---|
| Instalação do Dashboard falhou | Erro de disco cheio e espaço livre no grupo LVM | Ampliação do volume raiz |
| Notepad não aparecia como alerta genérico | Evento local existia; regra-base de criação de processo não equivalia a alerta visível | Criação de regra específica de teste |
| `100101` não aparecia | O evento estava classificado por `92057` | Encadeamento da regra personalizada com a nativa |
| Auditoria Linux sem resultado esperado | Chave personalizada ainda não mapeada | Inclusão de `wazuh_audit_test:write` na CDB |
| Input recebia, mas stream não mostrava eventos | Contador de entrada aumentava; stream estava parado | Ativação do stream e validação do roteamento |
| Eventos apareciam apenas em uma janela longa | Diferença de três horas no timestamp interpretado | Timezone do Input definido como `America/Sao_Paulo`; teste com evento novo |
| Simulador rejeitava JSON | Campo continha prefixo Syslog antes do JSON | Separação entre envelope e conteúdo JSON |
| Pipeline executava sem campos pesquisáveis | Necessidade de testar remoção de prefixo e extração separadamente | Regra final com `regex_replace`, `parse_json`, `select_jsonpath` e `set_fields` |

No caso do horário, é importante distinguir exibição de interpretação. Mostrar UTC ou horário local pode representar o mesmo instante. O erro relevante era interpretar o timestamp recebido em um fuso inadequado. A correção foi validada com mensagens novas; mensagens já indexadas não foram reprocessadas automaticamente.

## 8. Parser final e resultado

A mensagem recebida continha um envelope semelhante a `wazuh-siem ossec: {JSON}`. A regra final removeu o prefixo antes de interpretar o JSON e extraiu seis campos: ID, nível e descrição da regra; ID, nome e IP do agente.

A pipeline `Wazuh Processing` ficou conectada ao stream `Wazuh Alerts`. O Stage 0 passou a conter apenas `Parse Wazuh JSON Final`. As regras temporárias de debug foram removidas após a validação.

A consulta `wazuh_rule_id:100100` retornou um evento novo. Isso validou o caminho Windows → Sysmon → Wazuh → Graylog → campos estruturados. A ampliação do parser para MITRE e dados adicionais de processos foi proposta posteriormente, mas não integra o conjunto mínimo confirmado neste write-up.

## 9. Limites e evolução

O laboratório foi validado por testes funcionais pontuais. Não foram medidos throughput sustentado, perda de eventos, disponibilidade, retenção ou recuperação de desastre. O transporte UDP não fornece confirmação de entrega. O parser pressupõe o formato observado, incluindo o marcador `ossec:`.

Os casos Windows não demonstram detecção de toda execução PowerShell suspeita. O watch auditd foi configurado temporariamente e sua persistência após reboot não foi demonstrada. As referências de configuração publicadas precisam ser verificadas contra o ruleset e a versão de cada instalação.

Os próximos trabalhos são ampliar testes positivos e negativos, persistir a auditoria, avaliar transporte e retenção, adicionar campos úteis à investigação e produzir uma nova rodada de evidências exportáveis.
