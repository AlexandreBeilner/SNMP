- GET, GETNEXT, GETBULK, SET, WALK
- Diferença entre WALK e GETBULK e impacto em performance de coleta
- Polling: intervalos, escala e cuidado com carga no agente

## GET, GETNEXT, GETBULK, SET, WALK
O SNMP é um protocolo request-response. O network management system (NMS) envia uma requisição e o dispositivo remoto retornar uma resposta. No entanto, como o estado de um objeto pode mudar com o tempo (por exemplo, uma interface pode estar ativa agora, mas inativa em 5 minutos), o NMS consulta o dispositivo a cada poucos minutos para obter informações atualizadas. Isso é chamado de intervalo de sondagem.

#### GET
A mensagem GET recupera um valor OID específico. O NMS especifica o OID exato e o agente responde com o valor associado a esse OID. Por exemplo, se o NMS deseja o status da interface Ge0/1, ele usa o OID correspondente e a resposta do agente contém o status da interface.

#### GETNEXT
> **Ordem lexicográfica é ordem de dicionário.** A palavra `casa` vem antes de `casaco`, que vem antes de `caso`. Você compara letra por letra, da esquerda para a direita, e a primeira diferença decide tudo. Se uma palavra acabar antes e até ali era idêntica, a mais curta vem primeiro.

OID é a mesma coisa, com uma única troca: **cada "letra" é um número inteiro completo**, não um dígito. O ponto é só o separador entre as letras.

O algoritmo tem três passos:

1. Compare o 1º componente de A com o 1º de B, **como números inteiros**. Diferente? Acabou, quem for menor vem primeiro.
2. Iguais? Vá para o próximo componente e repita.
3. Um dos dois acabou e até ali era idêntico? O mais curto vem primeiro — é o "casa antes de casaco". É daqui que sai a regra do prefixo vir antes das extensões.
Essa seria uma lista ordenada de OIDs por exemplo:
```
1.2.2.2.3
1.2.2.2.3.5
1.2.2.2.3.5.3
1.2.2.2.3.5.6
1.2.2.2.3.6
```
É importante entender isso acima para saber como o GETNEXT funciona.
No GETNEXT nós vamos enviar um OID, ou uma lista de OIDs, ele pode ou não existir no destino e oque ele vai nos retornar é o proximo OID com base nessa ordem apresentada.
**Ou seja, o GETNEXT é utilizado para descobrir o que existe no agente, sem saber de antemão**

O mais comum de todos, de longe: **descobrir quantas interfaces um equipamento tem e como elas se chamam**.
Esse é o primeiro passo de todo sistema de monitoramento quando você cadastra um host novo.
Exemplo:
```
GETNEXT 1.3.6.1.2.1.2.2.1.2       →  1.3.6.1.2.1.2.2.1.2.1  = "lo"
GETNEXT 1.3.6.1.2.1.2.2.1.2.1     →  1.3.6.1.2.1.2.2.1.2.2  = "eth0"
GETNEXT 1.3.6.1.2.1.2.2.1.2.2     →  1.3.6.1.2.1.2.2.1.2.3  = "eth1"
GETNEXT 1.3.6.1.2.1.2.2.1.2.3     →  1.3.6.1.2.1.2.2.1.3.1  = 24
```

O primeiro OID enviado (`…2.2.1.2`, a coluna ifDescr sem índice) **não existe**, é um nó estrutural, sem valor. Serve só como ponto de partida. E a última resposta saiu do prefixo `…2.2.1.2` e caiu na coluna seguinte, que é o sinal de parada.

O resultado é a informação que você não tinha: **existem três interfaces, com ifIndex 1, 2 e 3**. Agora o sistema sabe quais índices existem e pode montar as consultas seguintes com GET direto (`ifOperStatus.2`, `ifHCInOctets.2`), que é o polling barato que roda a cada 5 minutos dali em diante.

Vale notar o padrão geral, porque ele se repete em tudo: a descoberta com GETNEXT é **esporádica** (uma vez no cadastro, depois a cada poucas horas para pegar mudanças), e a coleta com GET é **frequente**. Você não faz walk a cada ciclo de polling, seria desperdício de rede e de CPU do agente.

Vale lembrar que o GETNEXT não retorna apenas o OID que ele descobriu, ele retorna tambem os dados daquele OID.

#### GETBULK
Foi introduzido no SNMP v2.  Com uma unica requisição, ele pede varias próximas entradas de uma vez. Diferente do GETNEXT, que tem que retorna uma por vez. 
Tem dois parametros principais 
- **non-repeaters**: Quantas variáveis no início da lista devem ser tratadas como GetNext simples (uma resposta cada).
- **max-repetitions** Quantas vezes repetir o GetNext para as variáveis restantes, empacotando tudo numa única resposta. O `max-repetitions` só é aplicado aos OIDs que estão **depois** dos `non-repeaters` na lista, ou seja, os que sobraram para serem tratados como "repetidores". Na prática, esses costumam ser colunas de tabela, porque são os que fazem sentido "avançar N vezes".
Um exemplo de uso é: ao consultar uma tabela de interfaces, em vez de pedir uma interface por vez, pode pedir as proximas 15 numa unica chamada.
Resumindo o GETBULK é essencialmente "várias chamadas GETNEXT encadeadas, executadas numa única requisição/resposta". Você manda um OID ou uma lista de OIDs base e ele varre a árvore MIB a partir dali, retornando os próximos N OIDs (definidos pelo `max-repetitions`).
O fluxo tipoco é:
- Você não sabe os OIDs exatos de uma tabela (ex: lista de interfaces de um switch)
- Usa GETBULK a partir de um OID base (ex: `.1.3.6.1.2.1.2.2.1`- tabela de interfaces)
- Recebe de volta vários pares `OID → valor` já preenchidos

#### SET
O **SET** é a operação usada para **alterar/definir o valor** de um objeto gerenciado (OID) num agente SNMP. Diferente dos métodos acima o SET também escreve e modifica.
Ele serve para que o manager configure remotamente um dispositivo através do SNMP. Exemplos:
- Alterar o nome (sysName) de um equipamento.
- Habilitar/desabilitar uma porta de um switch.
- Reiniciar um processo ou dispositivo via um OID de controle.
- Ajustar parametros de configuração de um roteador.
Existem alguns requisito para o SET funcionar.
1. O objeto OID precisa ser gravavel. Na MIB cada objeto tem um atributo `MAX-ACCESS`, que pode ser:
   - `read-only` - Não aceita set
   - `read-write` aceita set
   - `read-create` aceita set e também criação de nova linha.
   - `not-accessible` - nem leitura nem escrita direta
1. Permissão de escrita no Agente.
   - Na v1/v2c depende da community string. É preciso que ela tenha permissão. 

#### SNMP WALK
SNMP WALK não é uma operação como as acima, ela é uma ferramenta.  Ela é implementada por exemplo no `snmpwalk`. Ela percorre uma arvore MIB utilizando repetidas chamadas GetNext (ou GetBulk em verções recentes).

Funciona da seguinte forma:
- Inicio: É fornecido um OID base. Assim como no getnext ou getbulk.
- GetNextRequest - O agente SNMP responde com o proximo OID na árvore, em ordem lexográfica, junto com seu valor.
- Repetição - O cliente pega esse OID retornado e utiliza como base para a proxima requisição.
- Parada - o processo continua até que o agente retorne um OID que esteja fora do sub-ramo original. Ou seja ao final do galho da arvore original. O agente também pode responder com `endOfMibView` isso também indica o final.
##### Diferença entre GetNext-based e GetBulk-based
- **SNMPv1**: o `snmpwalk` usa apenas `GetNextRequest`, uma requisição por vez, mais lento, mais tráfego de rede.
- **SNMPv2c/v3**: pode usar `GetBulkRequest`, que pede várias variáveis (linhas da tabela) em uma única requisição, tornando o walk muito mais eficiente, especialmente em tabelas grandes (ex: tabela de interfaces `ifTable`).
##### Casos de uso comuns
- **Descoberta de dispositivos**: levantar todas as interfaces (`ifTable`), suas descrições, status, tráfego (`IF-MIB`).
- **Inventário de MIBs suportadas**: ver o que um agente expõe.
- **Monitoramento**: coletar métricas em massa (memória, CPU, portas de switch) sem precisar saber os OIDs exatos de antemão.
- **Troubleshooting**: verificar se um agente está respondendo corretamente e o que está configurado.

### Diferença entre WALK e GETBULK e impacto em performance de coleta
Getbulk é um tipo de requisição do SNMP. Walk é uma ferramenta. Essa a principal diferença. O getbulk não subtitui o walk.
O walk é utilizado para perocorrer o ramo inteiro, inclusive, pode ser que ele utilize o GETBULK para isso. Em questão de performaçe, se o walk utilizar getBulk, ele tende a ser mais performatico que utilizar getNext.

Um **walk que usa GetBulk continua sendo um loop**.

### Polling: intervalos, escala e cuidado com carga no agente

Polling é o modelo de coleta. o manager NMS envia requisições `GET`, `GETNEXT` e `GETBULK` periodicamente para o agente e recebe valores das OIDs solicitadas. É o oposto do trap.

##### INTERVALOS DE COLETA
Não tem um valor único correto. Depende doque esta sendo medido.
- **Métricas de capacidade/tendência** (uso de CPU, memória, tráfego de interface para relatórios de longo prazo): 5 minutos é o padrão histórico (herdado do RRDtool/MRTG/Cacti). Suficiente para ver tendências sem gerar carga excessiva.
- **Métricas críticas para alarme** (link down, perda de pacotes, status de porta): pode-se ir a 1 minuto ou até 30s, mas aí SNMP puro já fica limitado, muita gente complementa com traps/syslog para eventos, e usa polling só para confirmação de estado.
- **Contadores de alta granularidade** (para detectar microbursts, picos curtos): polling não é a ferramenta certa. Nesses casos, sFlow/NetFlow/IPFIX ou telemetria por streaming são mais adequados, pois o SNMP não tem resolução temporal para isso.

Regra prática: quanto menor o intervalo, maior a carga cumulativa na rede e nos agentes, o ganho de "resolução" tem retorno decrescente rápido abaixo de 1 minuto para a maioria dos casos.

##### ESCALA
Problemas comuns quando o ambiente cresce:
- **Polling sequencial vs paralelo**: se o coletor pergunta a cada dispositivo um de cada vez, o tempo total de um ciclo pode ultrapassar o próprio intervalo configurado (ex.: 5 min de intervalo, mas o ciclo demora 7 min para percorrer todos os hosts). É preciso paralelizar (múltiplas threads/workers) e distribuir a carga ao longo do intervalo, não disparar tudo no mesmo instante ("thundering herd").
- **GETBULK em vez de múltiplos GET**: para tabelas (interfaces, ARP, rotas), usar `GETBULK` reduz drasticamente o número de round-trips comparado a `GETNEXT` iterativo.
- **Escalonamento de horários (jitter)**: se centenas de dispositivos são consultados no mesmo segundo exato, gera-se um pico de CPU/rede tanto no coletor quanto nos agentes. Espalhar o início das coletas (jitter aleatório) suaviza isso.
- **Polling distribuído**: em ambientes muito grandes, usar múltiplos coletores/pollers regionais (um por site, datacenter, ou segmento) em vez de um único coletor central falando com tudo via WAN.
- **Poller especializado por tipo de métrica**: separar coleta de métricas "pesadas" (tabelas grandes, como ARP em switches com milhares de MACs) das métricas "leves" (uptime, sysDescr), com intervalos diferentes para cada.

##### CUIDADOS COM CARGA NO AGENTE
O agente SNMP (o processo rodando no dispositivo monitorado) geralmente não é prioridade do hardware — ele compete por CPU com o plano de controle (roteamento, switching). Riscos:

- **CPU do agente**: em equipamentos mais antigos/simples (switches de borda, alguns roteadores), o agente SNMP pode consumir CPU de forma desproporcional, especialmente ao responder consultas de tabelas grandes (MIB de interfaces com centenas de portas, tabela de ARP grande, tabela de rotas BGP completa).
- **Impacto em produção**: já houve casos documentados de polling agressivo (intervalos curtos + `GETBULK` com `max-repetitions` alto) causando picos de CPU que degradam o plano de controle — em casos extremos, afetando convergência de protocolos de roteamento ou até derrubando sessões.
- **Boas práticas para mitigar**:
    - Ajustar `max-repetitions` do `GETBULK` para valores moderados (não pedir tudo de uma vez).
    - Evitar coletar tabelas inteiras com frequência alta; coletar só os índices/OIDs relevantes quando possível.
    - Usar SNMPv2c/v3 com `GETBULK` em vez de SNMPv1 (que só tem `GETNEXT`, muito mais ineficiente para tabelas).
    - Monitorar a própria carga que o polling gera nos dispositivos (muitos vendors expõem `sysUpTime`/CPU do próprio agente — ironicamente, via SNMP).
    - Ter rate limiting/circuit breaker no coletor: se um dispositivo está lento para responder, não insistir agressivamente nem empilhar requisições.