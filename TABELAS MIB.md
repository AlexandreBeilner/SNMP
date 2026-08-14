Um OID escalar representa um único valor.
Quando precisamos representar varias linhas de uma mesma informação, como as interfaces de rede de um computador, precisamos de um jjeito de listart, para cada inerface seu nome, seu tipo, sua velocidade, quantos bytes trefegam.
Por esse motivo, não podemos ter um único OID, porque qual interface isso representaria? A solução do SNMP é usar o próprio sufixo do OID como índice da linha.

### A estrutura conceitual

Pensa numa tabela normal, tipo planilha:

| Índice | ifDescr | ifType                | ifSpeed    |
| ------ | ------- | --------------------- | ---------- |
| 1      | lo      | 24 (softwareLoopback) | 10000000   |
| 2      | eth0    | 6 (ethernetCsmacd)    | 1000000000 |
| 3      | wlan0   | 71 (ieee80211)        | 866000000  |
Cada **coluna** (`ifDescr`, `ifType`, `ifSpeed`) tem seu próprio OID base fixo. Cada **linha** é identificada por um índice (1, 2, 3...). O SNMP "achata" essa planilha em uma lista de pares OID → valor, concatenando **OID_da_coluna + índice_da_linha**.
Ou seja. Essa tabela viraria o seguinte:
```
ifDescr.1 = lo
ifDescr.2 = eth0
ifDescr.3 = wlan0

ifType.1 = 24
ifType.2 = 6
ifType.3 = 71

ifSpeed.1 = 10000000
ifSpeed.2 = 1000000000
ifSpeed.3 = 866000000
```
À esquerda (conceitualmente) você pensa numa **tabela normal**: colunas com nomes, linhas numeradas. Mas o SNMP não transmite "tabelas", ele só sabe transmitir **pares OID → valor**, um de cada vez. Então ele "achata" essa tabela concatenando **OID da coluna + índice da linha**.

Se você quisesse "montar a linha 3 inteira" (índice `.3`), precisaria pegar manualmente cada valor, o SNMP não entrega isso agrupado, é você (ou a ferramenta de monitoramento) que precisa remontar isso depois.

### RESUMO
MIB é um esquema que diz o OID`1.3.6.1.2.1.1.5` se chama `sysName`, é do tipo STRING e significa X coisa. O Agente não armazena os dados na MIB. Ele guarda os dados onde faz sentido internamente (variaveis do kernel, arquivos, ...) e usa o MIB como um mapa de tradução. Pensar da seguinte forma: "quando alguém pedir um OID X, eu busco o valor Y na minha estrutura interna e devolvo formatado conforme o MIB manda".

Como a estrutura do MIB é uma arvore, ela possui nós estruturais e nós folha. Apenas os nós folha possuem valor. 

Os nós coluna falados anteriormente são nós que não possuem valor.  É um nó estrutural. A relação coluna tabela é definida estaticamente na MIB, através de uma estrutura hierárquica fixa de 3 níveis. Toda tabela SNMP segue exatamente esse padrão:
```
1.3.6.1.2.1.2.2        = ifTable        (nó estrutural: "isto é uma tabela")
1.3.6.1.2.1.2.2.1      = ifEntry        (nó estrutural: "isto é uma linha/registro")
1.3.6.1.2.1.2.2.1.2    = ifDescr        (coluna: "esta é a coluna nº2 da tabela")
1.3.6.1.2.1.2.2.1.2.1  = "lo"           (instância: linha 1, coluna ifDescr)
````

Repare no padrão: **Tabela → Entry (sempre sufixo `.1`) → número da coluna → índice da linha**. Isso é definido no arquivo MIB assim (simplificado):
```
ifTable OBJECT-TYPE ::= { interfaces 2 }              -- é uma tabela
ifEntry OBJECT-TYPE ::= { ifTable 1 }                  -- é a "linha padrão" dessa tabela
ifDescr OBJECT-TYPE ::= { ifEntry 2 }                  -- é a coluna 2 daquele Entry
    INDEX { ifIndex }                                  -- e o índice é definido pelo ifIndex
```