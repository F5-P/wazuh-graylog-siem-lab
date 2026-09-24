# Configurações de referência

[← Visão geral](../README.md)

Estes arquivos representam os trechos essenciais do laboratório. Foram reconstruídos e organizados para publicação; não são exportações integrais dos sistemas. Não substitua um `ossec.conf` existente por um destes fragmentos.

| Arquivo | Destino e uso |
|---|---|
| [local_rules.xml](wazuh/local_rules.xml) | Manager: incorporar as regras em `/var/ossec/etc/rules/local_rules.xml`, verificando IDs duplicados |
| [windows-sysmon.xml](wazuh/windows-sysmon.xml) | Agente Windows: adicionar o bloco `localfile` dentro de `ossec_config` |
| [linux-audit.xml](wazuh/linux-audit.xml) | Agente Linux: adicionar a fonte auditd dentro de `ossec_config` |
| [graylog-forwarding.xml](wazuh/graylog-forwarding.xml) | Manager: adicionar o bloco `syslog_output` dentro de `ossec_config` |
| [parse-wazuh-json.rule](graylog/parse-wazuh-json.rule) | Graylog: regra no Stage 0 de `Wazuh Processing`, conectada a `Wazuh Alerts` |

Os endereços `192.168.56.x` são endereços privados do laboratório. Adapte-os à sua topologia. O Input utilizado foi Syslog UDP na porta 5140, com timezone `America/Sao_Paulo` para o timestamp recebido sem fuso explícito no envelope.

Antes de reiniciar o Manager após alterações nas regras, execute nele:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
```

Esse comando verifica o carregamento das regras; não substitui um teste funcional com eventos. Confirme a presença da regra nativa `92057` na versão instalada antes de usar a regra filha `100101`.

O parser corresponde ao formato Syslog observado, com marcador `ossec:`. Ele extrai apenas os seis campos básicos validados. Não inclui a ampliação posterior proposta para MITRE, hashes ou linhas de comando, nem tratamento completo de mensagens malformadas.

Não foi incluído um Compose como instalador pronto: o objetivo desta publicação é documentar a implementação e suas detecções, sem apresentar uma reconstrução não testada da stack como implantação reproduzida.
