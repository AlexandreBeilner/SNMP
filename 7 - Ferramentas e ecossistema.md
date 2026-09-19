- net-snmp (snmpget, snmpwalk, snmptrap, snmptranslate)
- Bibliotecas Go (`gosnmp/gosnmp` vs. `go-snmplib`) — trade-offs já avaliados
- Integração com TSDB/observabilidade (Prometheus snmp_exporter, Zabbix, LibreNMS)

## net-snmp (snmpget, snmpwalk, snmptrap, snmptranslate) https://www.net-snmp.org/

O Net-SNMP é um conjunto de aplicações utilizado para implementar SNMP v1, SNMP v2c e SNMP v3 usando IPv4 e IPv6. O conjunto inclui:
- Aplicações de linha de comando para:
    - Recuperar informações de um dispositivo compatível com SNMP, seja usando solicitações únicas ( [snmpget](https://www.net-snmp.org/docs/man/snmpget.html) , [snmpgetnext](https://www.net-snmp.org/docs/man/snmpgetnext.html) ) ou múltiplas solicitações ( [snmpwalk](https://www.net-snmp.org/docs/man/snmpwalk.html) , [snmptable](https://www.net-snmp.org/docs/man/snmptable.html) , [snmpdelta](https://www.net-snmp.org/docs/man/snmpdelta.html) ).
    - manipular informações de configuração em um dispositivo compatível com SNMP ( [snmpset](https://www.net-snmp.org/docs/man/snmpset.html) ).
    - recuperar uma coleção fixa de informações de um dispositivo compatível com SNMP ( [snmpdf](https://www.net-snmp.org/docs/man/snmpdf.html) , [snmpnetstat](https://www.net-snmp.org/docs/man/snmpnetstat.html) , [snmpstatus](https://www.net-snmp.org/docs/man/snmpstatus.html) ).
    - Converter entre formas numéricas e textuais de OIDs MIB e exibir conteúdo e estrutura MIB ( [snmptranslate](https://www.net-snmp.org/docs/man/snmptranslate.html) ).

## Integração com TSDB/observabilidade (Prometheus snmp_exporter, Zabbix, LibreNMS)

#### Prometheus + snmp_exporter
O Prometheus não fala SNMP nativamente. Ele usa um **exporter** intermediário que traduz SNMP → métricas Prometheus.

````
Prometheus → scrape HTTP → snmp_exporter → SNMP GET/WALK → dispositivo
````

O `snmp_exporter` recebe do Prometheus uma requisição HTTP do tipo:
````
GET /snmp?module=if_mib&target=192.168.1.1
````

e ele mesmo faz as chamadas SNMP para o `target`, convertendo os OIDs em métricas no formato Prometheus.

**Configuração básica (`prometheus.yml`):**
```yaml
scrape_configs:
  - job_name: 'snmp'
    static_configs:
      - targets:
          - 192.168.1.1
          - 192.168.1.2
    metrics_path: /snmp
    params:
      module: [if_mib]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: snmp-exporter:9116
```

**Ponto-chave:** o `snmp.yml` do exporter (arquivo de configuração de módulos) precisa ser gerado a partir das MIBs dos seus equipamentos. Existe uma ferramenta `generator` que compila MIBs em módulos SNMP prontos para scrape, isso costuma ser o passo mais trabalhoso.

**Vantagens:**
- Se integra ao ecossistema Prometheus/Grafana já existente
- PromQL para queries flexíveis
- Bom para séries temporais de alta cardinalidade

**Desvantagens:**
- Não faz descoberta automática de dispositivos (você define os targets)
- Configuração de módulos SNMP customizados dá trabalho
- Sem gestão de traps/alertas nativa (precisa de `snmptrapd` + algo para converter em alertmanager)

#### Zabbix
Zabbix tem **suporte nativo a SNMP** embutido no servidor/proxy, sem exporter externo.

**Como funciona:**
- Cada "item" monitorado pode ser do tipo "SNMP agent"
- Você configura o OID direto na interface, ou usa templates prontos (Zabbix tem uma biblioteca grande de templates para switches Cisco, HP, etc.)
- Suporta **SNMP discovery** (LLD - Low Level Discovery): descobre automaticamente interfaces de rede, discos, sensores, e cria itens/triggers dinamicamente
- Suporta traps via `snmptrapd` integrado ao Zabbix trapper

Fluxo tipico
```
Zabbix Server/Proxy → polling SNMP periódico → dispositivo
                    ↕
              Templates + LLD (auto-descoberta de interfaces)
```

**Vantagens:**
- Tudo integrado: coleta, alertas, dashboards, sem precisar orquestrar múltiplos serviços
- LLD é muito forte para redes com muitas interfaces variáveis
- Bom suporte a triggers complexos e escalonamento de alertas

#### LibreNMS
É a opção mais "pronta para uso" quando o foco é **especificamente rede**.

**Como funciona:**
- Auto-discovery agressivo: você só aponta o IP/range e ele detecta o tipo de dispositivo (via sysObjectID) e aplica automaticamente os OIDs certos
- Tem um banco enorme de definições para milhares de modelos de equipamentos (Cisco, Juniper, MikroTik, Ubiquiti, etc.)
- Inclui gráficos, mapas de topologia (via CDP/LLDP), alertas, e até uma API REST
- Também tem integração para exportar métricas para o Prometheus (`librenms-agent` ou via API)

**Vantagens:**
- Setup extremamente rápido para monitoramento de rede tradicional
- Melhor "out of the box" para device discovery e billing/topologia
- Menos configuração manual de OIDs

**Desvantagens:**
- Menos flexível para métricas não-relacionadas a rede
- Escala pior que Prometheus em ambientes muito grandes (milhares de dispositivos com alta frequência de coleta)