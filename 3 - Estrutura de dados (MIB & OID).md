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

A lógica de fundo é a **mesma** que vimos até agora (SMI, OID, OBJECT-TYPE, árvore hierárquica), não é um sistema paralelo.
O que muda é que MIBs proprietárias vivem sobre um ramo diferente das MIB padrões que vimos. Um local que é reservado expecificamente para isso.
````
1.3.6.1.4.1  → iso.org.dod.internet.private.enterprises
````

Esse ramo (`enterprises`) é administrado pela IANA, que atribui **um número único de "Private Enterprise Number" (PEN)** para cada empresa que solicita. Exemplos reais:
```
1.3.6.1.4.1.2011   → Huawei
1.3.6.1.4.1.3902   → ZTE
1.3.6.1.4.1.5875   → FiberHome
```

A partir desse número, cada fabricante é **livre para criar sua própria sub-árvore como quiser**, sem precisar de aprovação de ninguém, sem RFC, sem padronização externa. É por isso que a estrutura interna varia muito de fabricante pra fabricante.
Nas MIBs padrão existe um objeto que todo mundo segue igual. Nas MIBs proprietárias cada fabricante escreve, documenta e mantém os arquivos, logo não existe padrão. Com isso temos:
- Menos padronização entre fabricantes (Huawei e ZTE vão modelar "status de uma ONU" de formas completamente diferentes)
- Você depende de **conseguir o arquivo MIB do fabricante** (geralmente via portal de suporte, contato com o fabricante, ou às vezes só encontrando em fóruns/repositórios da comunidade)

#### Leitura e compilação
O termo "Compilar uma MIB" significa pegar aquele arquivo de texto (escrito em sintaxe SMI/ASN.1) e fazer seu software de gerenciamento **entender e carregar** essa definição, pra que ele consiga traduzir OIDs numéricos em nomes legíveis e saber interpretar os tipos corretamente.
O passo a passo para acesso a um MIB proproetário é:
1. Obter o arquivo MIB do fibricante - Ex: `HUAWEI-OLT-MIB.mib`, `ZTE-GPON-MIB.mib`. Geralmente vem em pacotes com várias MIBs interdependentes.
2. Resolver dependencias - MIBs proprietárias quase sempre **importam definições de outras MIBs** (via `IMPORTS` no início do arquivo), tanto MIBs padrão (como `SNMPv2-SMI`, `SNMPv2-TC`) quanto outras MIBs do próprio fabricante. Se faltar um arquivo dependente, a compilação falha.
3. Compilar/carregar no software de gerenciamento - Ferramentas comuns (net-snmp, Zabbix, PRTG, MIB Browser, iReasoning, etc) têm um mecanismo de "carregar MIB" que faz o parsing do arquivo e valida a sintaxe. Se der certo, o software passa a "conhecer" aqueles OIDs e consegue:
	- Fazer `SNMP WALK` mostrando nomes em vez de números crus
	- Interpretar corretamente tipos, enums e unidades
4. Testar/validar - Rodar um `snmpwalk` apontando pro ramo do fabricante pra conferir se os valores batem com o esperado.

### Resumo direto

|Aspecto|MIB padrão (RFC)|MIB proprietária (fabricante)|
|---|---|---|
|Localização na árvore|`1.3.6.1.2.1` (mib-2)|`1.3.6.1.4.1.<PEN>` (enterprises)|
|Quem define|IETF via RFC|O próprio fabricante|
|Padronização entre marcas|Alta (mesmo objeto = mesmo OID sempre)|Nenhuma (cada fabricante modela do seu jeito)|
|Onde conseguir o arquivo|Público, disponível em repositórios IETF/IANA|Portal de suporte do fabricante, às vezes restrito|
|Complexidade típica|Estrutura mais simples e genérica|Tabelas profundas e específicas (ex: hierarquia OLT→PON→ONU)|

## Mapeamento de OIDs relevantes para ONUs: status, sinal óptico (Rx/Tx power), uptime, etc.

#### GPON
**GPON (Gigabit Passive Optical Network)** é uma tecnologia de rede óptica passiva, padronizada pela ITU-T na série **G.984** (e evoluções: G.987 para XG-PON, G.9807 para XGS-PON). É a tecnologia por trás da maioria das redes de fibra até a casa do cliente (FTTH) hoje em dia.

Aqui temos um ponto que ja foi levanteado anteriormente, a GPON não usa SNMP para gerencias a ONU. A comunicação OLT <-> ONUpara gerencimanto é feita via um protocolo chamado OMCI **(ONT Management and Control Interface)**, definido pelo padrão ITU-T G.988. O OMCI é um modelo padronizado, e não um protocolo proprietário, o que o torna a base para redes de acesso multi-fabricante onde a OLT, as ONUs e o sistema de gerenciamento de rede podem vir de fornecedores diferentes.
Mesmo exitindo uma padronização entre OLT -> ONU, ao buscar os dados da OLT via SNMP não existe um padrão de resposta.
Quando a OLT recebe os dados via OMCI (protocolo padronizado, camada interna OLT↔ONU) e precisa **expor isso via SNMP** para o seu NMS (camada externa, northbound), o fabricante faz essa "tradução" **do jeito que ele quiser**.

|O que pode variar entre fabricantes|Exemplo|
|---|---|
|**Estrutura da tabela/indexação**|Um fabricante indexa ONU por `slot.porta.onuId`, outro por um índice calculado único|
|**Unidade/escala do valor**|Potência óptica pode vir em dBm × 100 (centésimos) num fabricante, e × 10 (décimos) em outro|
|**Granularidade**|Um fabricante expõe só "potência atual", outro expõe também histórico/mín/máx|
|**Quais campos existem**|Um fabricante expõe "distância estimada da fibra" via SNMP, outro só disponibiliza isso via CLI/OMCI direto, sem espelhar no SNMP|
|**Nomenclatura e organização da MIB**|Nomes de objetos e agrupamento em tabelas totalment|

#### OIDs relevantes
### Huawei (PEN 2011)

Ramo base: `1.3.6.1.4.1.2011`

|Informação|OID|Observação|
|---|---|---|
|Status da porta PON (online/offline)|`1.3.6.1.4.1.2011.6.128.1.1.2.21.1.10` (`hwGponDeviceOltControlStatus`)|1 = Online, 2 = Offline|
|Status Ethernet da ONU online|`1.3.6.1.4.1.2011.6.128.1.1.2.62.1.22` (`hwGponDeviceOntEthernetOnlineState`)|1 = Online, 2 = Offline|
|Potência Rx da ONU (vista pela OLT)|`1.3.6.1.4.1.2011.6.128.1.1.2.51.1.4`|Valor retornado em formato como INTEGER: -1311, sendo necessário dividir por 100 para obter o valor real em dBm (ex: -13,11 dBm) [IT Blog](https://ixnfo.com/en/oid-and-mib-for-huawei-olt-and-onu.html)|
|Potência Rx na OLT (sinal vindo da ONU)|`1.3.6.1.4.1.2011.6.128.1.1.2.51.1.6`|Referenciado como OLTRX na estrutura SNMPv2-SMI::enterprises.2011.6.128.1.1.2.51.1.6 [GPON Solution](https://gponsolution.com/snmp-mib-huawei-olt-ont.html)|
|Temperatura óptica da ONU|`1.3.6.1.4.1.2011.6.128.1.1.2.51.1.1` (`hwGponOntOpticalDdmTemperature`)|—|
|Distância estimada até a ONU|`1.3.6.1.4.1.2011.6.128.1.1.2.46.1.20` (`hwGponDeviceOntControlRanging`)|—|
|Quantidade de MACs conectados à ONU|`1.3.6.1.4.1.2011.6.128.1.1.2.46.1.21` (`hwGponDeviceOntControlMacCount`)|
O status de link das portas GPON (Online = 1, Offline = 2) fica em hwGponDeviceOltControlStatus, sob 1.3.6.1.4.1.2011.6.128.1.1.2.21.1.10. A indexação dessas tabelas segue o padrão `oid.porta_da_olt.id_da_ont`, ou seja, cada valor individual de uma ONU específica é acessado concatenando o OID base com o índice da porta e o ID da ONU.

### FiberHome (PEN 5875)

Ramo base: `1.3.6.1.4.1.5875.800`

|Informação|OID|Observação|
|---|---|---|
|Status da ONU|`1.3.6.1.4.1.5875.800.3.10.1.1.11.<índice>`|Nos modelos 5116: 0 = offline/fibra cortada, 1 = online, 2 = corte de energia. Nos modelos 5516: 0 = fibra cortada, 1 = online, 2 = corte de energia, 3 = offline [PDFCOFFEE.COM](https://pdfcoffee.com/fiberhome-gepon-5116-5516-mib-open-interface-specifications-pdf-free.html)|
|Número do slot|`1.3.6.1.4.1.5875.800.3.10.1.1.2`|Slot number, tipo Int [PDFCOFFEE.COM](https://pdfcoffee.com/fiberhome-gepon-5116-5516-mib-open-interface-specifications-pdf-free.html)|
|Número da porta PON|`1.3.6.1.4.1.5875.800.3.10.1.1.3`|PON number [PDFCOFFEE.COM](https://pdfcoffee.com/fiberhome-gepon-5116-5516-mib-open-interface-specifications-pdf-free.html)|
|Número da ONU|`1.3.6.1.4.1.5875.800.3.10.1.1.4`|ONU number/índice [PDFCOFFEE.COM](https://pdfcoffee.com/fiberhome-gepon-5116-5516-mib-open-interface-specifications-pdf-free.html)|
|Status geral do OLT (fonte de energia)|`1.3.6.1.4.1.5875.800.3.60.1.1.2.<n>`|1 = normal, 2 = baixa voltagem, 3 = falha [PDFCOFFEE.COM](https://pdfcoffee.com/download/oids-fiberhome-pdf-free.html)|
|Temperatura|`1.3.6.1.4.1.5875.800.3.9.4.5.0`|—|
**Fórmula de indexação da ONU (importante):** o índice é calculado como (slot) × 2²⁵ + (PON) × 2¹⁹ + (ONU) × 2⁸ + porta, bem diferente do esquema mais simples da Huawei.