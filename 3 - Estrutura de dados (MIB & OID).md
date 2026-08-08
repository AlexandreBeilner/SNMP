- SMI (Structure of Management Information) e tipos de dados
- Árvore de OIDs, MIB-II (RFC 1213) e MIBs padrão
- Leitura e compilação de MIBs proprietárias (ex.: MIBs de fabricantes de OLT — Huawei, ZTE, Fiberhome)
- Mapeamento de OIDs relevantes para ONUs: status, sinal óptico (Rx/Tx power), uptime, etc.

## SMI
O SMI é conjunto de regras que define como as infomações devem ser nomeadas, estruturadas e representadas. Ele define a sintaxe utilizada para descrever cada objeto gerenciável.
Ele serve para definir como podemos nomear um objeto gerenciado, como definir os tipos/sintaxe desse objeto e como codificar esses dados para a transmissão na rede.

A versão 1 do SNMP utiliza o SMIv1, a versão 2 e 3 utiliza a SMIv2, que trouxe mais tipos e expressividade.

Ele possui 3 componentes principais sendo:
1. **Definição de módulo** - Como um módulo MIB é estruturado e documentado. Utilizando a macro `MODULE-IDENTITY`
2. **Definição de objetos** - Como um objeto gerenciado individual é descrito. Cada objeto tem Nome, Sintaxe/Tipo de dado, Modo de acesso, Status e Descrição textual. É feito utilizando o `NOTIFICATION-TYPE`
3. **Definição de notificações** - Como traps/notifications são definidas — usando `NOTIFICATION-TYPE` (SMIv2) ou `TRAP-TYPE` (SMIv1).

#### Tipos de dados
Ele possui alguns tipos primitivos
- `INTEGER` - Número inteiro
- `OCTET STRING` - Sequência de bytes (usado para strings, dados binários)
- `OBJECT IDENTIFIER (OID)` - Identificador hierárquico único de um objeto na árvore MIB.
- `NULL` - Ausência de valor
E possui tipos que são construidos em cima dos primitivos, mais especificos para gerenciamento de rede.
- `Integer32`Inteiro de 32 bits com sinal
- `Unsigned32`Inteiro de 32 bits sem sinal (SMIv2)
- `Counter32`Contador que só incrementa, "dá a volta" (wrap) ao chegar no máximo, ex: pacotes recebidos
- `Counter64`Igual ao Counter32, mas 64 bits, pra links de alta velocidade onde 32 bits estoura rápido (SMIv2)
- `Gauge32`Valor que pode subir e descer, mas fica "travado" nos limites mín/máx, ex: uso de banda atual
- `TimeTicks`Tempo decorrido, medido em centésimos de segundo (ex: uptime)
- `IpAddress`Endereço IPv4 (4 bytes)
- `Opaque`Dado arbitrário, "encapsulado" sem interpretação direta pelo SNMP
#### Diferença importante: SMIv1 vs SMIv2

- SMIv1 não tinha `Unsigned32` nem `Counter64`, foram adicionados no SMIv2 justamente porque redes mais rápidas estouravam contadores de 32 bits rapidamente demais.
- SMIv2 formalizou melhor a sintaxe das macros (`OBJECT-TYPE`, `MODULE-IDENTITY`, etc).

##  Árvore de OIDs, MIB-II e MIBs padrão
#### Arvores OIDs
As arvore OIDs são a estrutura/organização geral. É uma árvore hierárquica global. Cada nó tem um numero e pode ter um nome. Todo objeto gerenciavel do mundo SNMP tem um endereço único na arvore. 
Ela não armazena valores, armazena identificadores (endereços/nomes) para cada tipo de informação que pode ser gerenciada em um dispositivo.
Essa arvore é como um índice/esquema de endereçamento, não um banco de dados. Os valores reais ficam armazenados no Agente SNMP do dispositivo. A árvore só define **onde** cada tipo de informação "mora" e **como ela se chama** de forma padronizada e única no mundo inteiro.

Um **OBJETO GERENCIAVEL** que foi falado algumas vezes é qualquer caracteristica/informação de um dispositivo que pode ser monitorada ou configurada via SNMP. Exemplo:
- O numero de pacotes recebidos em uma interface
- O status up/down de uma interface
- A tabela de rotas IP

Cada um desses conceitos é definido em uma MIB, com um tipo de dados, via SMI como visto e um OID único.
Cada objeto gerenciavel recebe um número OID que é global e nao se repete em nenhum outro lugar do mundo. Isso é o endereço único.
Exemplo:
- 1.3.6.1.2.1.1.3.0 -> sysUpTime
O OID possui duas partes conceituais:
- A parte que identifica o tipo do objeto, no exemplo acima é `1.3.6.1.2.1.1.3` = isso é um sysUpTime
- O ultimo valor `.0` diz qual o numero da interface.

Isso é importante pois pense no seguinte
Um roteador Cisco, um switch Juniper e um servidor Linux — todos, se implementarem MIB-II corretamente, vão usar o **mesmo OID** (`1.3.6.1.2.1.1.3.0`) para representar "uptime". Isso significa que seu sistema de monitoramento (Zabbix, PRTG, Nagios, o que for) pode perguntar o mesmo OID para equipamentos completamente diferentes, de fabricantes diferentes, e obter a informação equivalente.
Quando seu NMS faz um `GET` ou `WALK` via SNMP, ele está literalmente perguntando "me dá o valor que está armazenado sob este OID". Sem um endereçamento único e padronizado, o protocolo inteiro simplesmente não funcionaria, não haveria como pedir uma informação específica de forma confiável.

#### MIB-II
O MIB-II É a definição concreta de um conjunto padronizado de objetos, localizado no ramo `1.3.6.1.2.1` (`iso.org.dod.internet.mgmt.mib-2`) da árvore. MIB-II define grupos como `system`, `interfaces`, `ip`, `icmp`, `tcp`, `udp`, etc — e cada objeto dentro desses grupos tem seu OID exato dentro dessa estrutura.

**`1.3.6.1.2.1` é o "endereço-raiz" reservado oficialmente para o MIB-II** (o nó chamado `mib-2`). Isso foi definido pela IANA/IAB como o local padrão onde toda a árvore de objetos do MIB-II fica pendurada. Nenhum outro conjunto de objetos pode usar esse mesmo ramo.

Os numeros que vem depois desse endeerço base indicam o grupo de informações. Por exemplo:
```
1.3.6.1.2.1.1   → system      (grupo 1: informações gerais do sistema)
1.3.6.1.2.1.2   → interfaces  (grupo 2: informações de interfaces de rede)
1.3.6.1.2.1.3   → at          (grupo 3: address translation - legado)
1.3.6.1.2.1.4   → ip          (grupo 4: informações de IP)
1.3.6.1.2.1.5   → icmp        (grupo 5: estatísticas ICMP)
1.3.6.1.2.1.6   → tcp         (grupo 6: estatísticas TCP)
1.3.6.1.2.1.7   → udp         (grupo 7: estatísticas UDP)
1.3.6.1.2.1.11  → snmp        (grupo 11: estatísticas do próprio SNMP)
```

E dentro de cada grupo, os números continuam se ramificando para identificar cada objeto individual. Por exemplo, dentro do grupo `system` (`1.3.6.1.2.1.1`):
```
1.3.6.1.2.1.1.1  → sysDescr     (descrição do sistema)
1.3.6.1.2.1.1.3  → sysUpTime    (tempo de atividade)
1.3.6.1.2.1.1.5  → sysName      (hostname)
```

#### MIBs padrão
É um conteito mais aplo, MIB-II é uma MIB especifica. Existem centas de outras MIBs padronizadas e também MIBs proprietarias de fabricantes que ocupam outros ramos da arvore. 

## Leitura e compilação de MIBs proprietárias (ex.: MIBs de fabricantes de OLT — Huawei, ZTE, Fiberhome)