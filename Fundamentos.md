## [[Fundamentos]]

**SNMP é Simple Network Management Protocol**, ele é um protocolo de gerenciamento de rede que é utilizado para gerenciar (controlar e monitorar) dispositivos de infraestrutura de rede (roteadores, switches, ONUs, servidores, etc...).

Um sistema completo SNPM consiste em 3 partes
1. Manager (gerenciador)
2. Agent (Agente)
3. Manager information base (MIB) (Base de Informações Gerenciais)

#### SNMP Manager

O SNMP Manager pode solicitar (polls) informações dos dispositivos gerenciados por SNMP ou Agents (roteadores, switches, servidores de rede etc.).

O SNMP Manager também pode receber informações não solicitadas, conhecidas como "traps", de um dispositivo gerenciado pelo SNMP (roteadores, switches, servidores de rede etc.).

O SNMP Manager é frequentemente chamado de Network Management Station (NMS).

#### SNMP Agent

O SNMP Agent é o software cliente SNMP que é executado em um dispositivo gerenciado por SNMP, como um roteador, um switch ou um servidor.
Todos os tipos de dados são coletados pelo próprio dispositivo e suas atividades são armazenadas em um banco de dados local chamado Base de Informações de Gerenciamento (MIB).
O agente pode então responder  *SNMP pools* com informações do banco de dados e pode enviar alertas não solicitados ou "traps" para um gerenciador SNMP.
O agente executa 3 ações em relação ao manager. Sendo elas
1. Responder requisição: Quando o manager solicita na porta UDP 161, o agente processa a requisição, que pode ser um comando de leitura ou um comando de escrita/alteração de configuração.
2. Traps: Dispara notificações de eventos para a porta 162 do Manger de forma unidirecional, sem controle de entrega.
3. Informs: Dispara notificações de eventos para a porta 162 do Manager, aqui ele espera receber uma mensagem de confirmação (ACK) do manager. Caso ele não receba a confirmação, ele reenvia a evento.

#### Management Information Base MIB

Base de informação de gerenciamento e um database que contém coleções de informações organizadas hierarquicamente em estrutura de arvore.
Na verdade, todo o MIB é uma coleção de variáveis ​​(OIDs) armazenadas em MIBs individuais mais granulares, que formam os ramos da árvore.
Cada MIB é baseado na linguagem Abstract Syntax Notation 1 (ASN.1).
![[Pasted image 20260728173223.png|587]]


A comunicação ocorre via protocolo UDP. A porta 161 é utilizada pelo SNMP Manager para solicitar (poll) MIB dos SNMP agents. E os SNPM agents utilizam a porta 162 para enviar traps ou informações solicitadas para o SNMP Manager.
O agent escuta passivamente a porta 161 e o NMS (SNMP Manager) escuta a 162.

![[Pasted image 20260728173703.png]]

Resumindo a comunicação, o NMS vai enviar solicitações via UDP para a porta 161, o Agente vai receber essa solicitação. Ele então processa a solicitação e cria uma resposta que é enviada via para o NMS.
Para as Traps e Informes que são enviadas pelo agente a comunicação é ao contrario. O NMS vai ouvir passivamente a porta UDP 162, e o Agent vai enviar mensagens para a porta 162.

**Polling( Manager -> Agent )**: Requisição vai para a porta de destino 161
**Trap ( Agent -> Manager )**: Notificação vai para a porta de destino 162

O protocolo UDP não formaliza uma conexão entre quem envia e quem recebe, dessa forma, ele acaba sendo mais rápido, mas em contrapartida, pode ser que ocorra a perda de pacotes na comunicação.

#### ALTERNATIVAS AO SNMP
O SNMP hoje é considerado legado, comparado a novos que surgiram. Podem é amplamente suportado, e bom para monitoramento de equipamentos simples.

As principais alternativas ao SNMP são:
1. NETCONF (Network Configuration Protocol): Protocolo IETF criado para superar o SNMP em configuração. Utiliza modelos de dados YANG (Organiza os dados hierarquicamente em estrutura de arvore) e codificação em XML. Suas principais aplicações são para automação, provisionamento e alterções programáticas da configuração de switches e roteadores de forma padronizada. É ideal para gerenciamento de configurações e segurança transacional visto que suporta operações de commit e rollback, garantindo que um bloco de configurações seja aplicado integralmente (ou desfeito em caso de erro), evitando deixar a rede em estado inconsistente.
2. gNMI (gRPC Network Management Interface): Interface padronizada pelo OpenConfig (Define e implementa uma camada de software comum e independente de fornecedores para gerenciar dispositivos de rede.), unificando a configuração e monitoramento de rede utilizando gRPC. Os dados são no modelo YANG mas serializados em protobuffers. É utilizado para configuração de dispositivos modernos de rede e coleta de telemetria. Ideal para quando precisa de alto desemprenho.
3. Telemetria Streaming: Esse é um paradigma de coleta de dados onde o equipamento de rede envia os dados continuamente para o coletor com base em assinaturas, sem precisar ficar solicitando. Esse é utilizado para observabilidade avançada de rede.


----
**Arquitetura de Proxy OMCI** ONUs não são monitoradas diretamente via rede IP pelo sistema de gerência. O servidor realiza requisições SNMP para a OLT; a OLT, por sua vez, comunica-se com a ONU via protocolo OMCI (Optical ONT Management and Control Interface) na camada física PON. O SNMP é o padrão estabelecido para a OLT expor os dados coletados do OMCI para camadas superiores.

**Baixo Overhead Computacional** Uma única OLT gerencia de milhares a dezenas de milhares de ONUs simultaneamente. O control plane da OLT possui restrições de CPU e memória. O SNMP utiliza transporte UDP sem estado e codificação ASN.1/BER, exigindo processamento mínimo. Protocolos como NETCONF (parsing de XML em sessões TCP/SSH) ou gNMI (gRPC/HTTP2) esgotariam os recursos da OLT ao tentar exportar dados em larga escala.

**Estrutura de Dados Hierárquica (MIBs)** O modelo de dados baseado em árvores OID (Object Identifiers) do SNMP mapeia de forma nativa e exata a topologia de hardware de redes ópticas passivas. O endereçamento flui logicamente através de índices: `Chassi > Slot > Porta PON > ID da ONU > Porta Ethernet (UNI)`.

**Eficiência no Envio de Alarmes Físicos (Dying Gasp)** Quando a energia elétrica de uma ONU é cortada, capacitores internos garantem milissegundos de energia para enviar um último pulso óptico (Dying Gasp) informando a queda. A OLT recebe o sinal físico e gera instantaneamente um SNMP Trap (datagrama único UDP, fire-and-forget). A tentativa de estabelecer um 3-way handshake TCP (exigido por NETCONF ou Telemetria) para notificar o sistema geraria latência excessiva ou falharia.

**Ausência de Suporte de Hardware** Fabricantes globais de equipamentos de acesso (Huawei, ZTE, FiberHome, Nokia) desenvolvem chipsets e hardwares otimizados há décadas para o processamento de SNMP no hardware de gerência. Suporte a gNMI ou Telemetria Streaming em OLTs comerciais ou switches de última milha é técnica e comercialmente inexistente na infraestrutura atual.

**Topologia SNMP**

- **SNMP Manager:** O seu servidor (sistema de monitoramento, script em background, Zabbix, etc.).
    
- **SNMP Agent:** O sistema operacional rodando na controladora da OLT.
    

**Fluxo de Notificação (ONU Offline via Trap)**

1. **Evento Físico:** A ONU perde a conexão (corte de fibra na rua, desconexão física, ou queda de energia gerando um pulso "Dying Gasp").
    
2. **Detecção OLT:** A placa PON da OLT identifica a perda de sincronismo (LOS - Loss of Signal) ou recebe o Dying Gasp.
    
3. **Geração do Trap:** O Agent SNMP da OLT monta o datagrama de Trap contendo o OID específico do evento (ex: `onuOffline`) e os índices de identificação da ONU (Slot/Porta/ID).
    
4. **Envio:** A OLT dispara o Trap via UDP para a porta 162 do IP do seu Manager.
    
5. **Processamento:** O seu Manager recebe o Trap e altera o estado da ONU no dashboard instantaneamente, sem precisar aguardar o próximo ciclo de leitura (polling).