- SNMPv1, v2c e v3: diferenças práticas
- Limitações de segurança do v1/v2c (community string em texto puro)
- SNMPv3: USM (autenticação MD5/SHA, criptografia DES/AES) e VACM (controle de acesso)

#### Community strings ( Oque são )
É uma forma simples de atenticação, é como se fosse uma senha em texto puro que é enviada do sistema gerenciador para um agente. Isso da acesso aos dados do dispositivo. Quando o sistema gerenciador quer cosnultar um agente ele precisa incluir a community string correta na mensagem.
Normalmente existem pelo menos 2 tipos configurados no dispositivo. Uma Read-Only utilizada para consultar (GET) informações e outra Read-Write que alem de consultar permite alterar (SET) informações.
Isso é um problema de segunça na V1 e V2 pois a community string trafega como texto puro na rede. Sem nenhuma criptografia. Ou seja qualquer pessoa capturando o trafego (Com Wireshark por exemplo) consegue ver a string, não á autenticação real, apenas uma comparação de string = string.

#### SNMP v1
Essa é a primeira versão do SNMP, requer apenas uma string de comunidade em texto simples (**plain-text community string**) para autenticação de pacotes e restrição de acesso. Em outras palavras, o SNMPv1 utiliza strings de comunidade de leitura e gravação e somente leitura. Esse tipo de uso é vulnerável a ataques de rede, pois não há criptografia na transferência de dados. Basicamente, apresenta limitações de desempenho e segurança.
Ele suporta apenas contadores de 32 bits.
As operações do protocolo nessa versão são:
- GET: Recupera informações de um dispositivo
- GETNEXT: Obtém o próximo pedaço de dados na sequencia. 
- SET: Modifica configurações.
- TRAP: Recebe alerta dos dispositivos. 

A _community string_ funciona como uma senha estática pré-compartilhada para autenticação entre o SNMP Manager e o SNMP Agent. O Agent só processa requisições ou envia Traps se a string do pacote coincidir com a configurada em sua memória.

**Mecânica do Texto Puro (Plain-text)** O termo _plain-text_ indica a total ausência de criptografia ou funções de hash. A senha é inserida diretamente no cabeçalho do pacote SNMP e transmitida via UDP pela rede. Qualquer interceptação de tráfego (ex: via Wireshark ou tcpdump) revela a credencial imediatamente.

Em resumo, é fácil de configurar, é pouco eficiente, suporta contador 32 bits, pouco segura e possui as funcionalidades de GET, GETNEXT, SET e TRAP.

#### SNPMv2c

É a segunda versão do SNMP e a mais utilizada. Ele resolve as limitações do SNMPv1 e oferece melhor desempenho e tratamento de erros mais eficiente. O SNMPv2c utiliza strings de comunidade SNMP de leitura e gravação, e somente leitura. Com a opção somente leitura, permite o acesso a objetos da Base de Informações de Gerenciamento (MIB) apenas para leitura, enquanto com as strings de comunidade de leitura e gravação, os usuários podem editar e fazer alterações, como mudanças de configuração. É mais seguro que a versão 1, mas não tão seguro quanto a versão 3. O SNMPv2c também é vulnerável a ataques.
Na versão 2 os comandos de GET, GETNEXT e SET são identicos a V1. A principal vantagem desse comparado a v1 é que ela adiciona o comando INFORM. Diferente de uma TRAP, os INFORMs são confirmados positivamente com uma mensagem de resposta. Se um gerenciador não responder a um Inform, o agente SNMP reenviará o Inform.

Operações adicionadas nessa versão:
- GETBULK: Recuperar grandes quantidades de dados de forma eficiente.
- INFORM: Receba a confirmação das mensagens enviadas.
- TRAP (melhorado): Mensagens de trap aprimoradas com informações adicionais.

Em resumo e facil de fazer o setup, e pouco eficiente, ja suporta 64 bits, e possui as funcionalidades GET, GETBULK, GETNEXT, SET, INFORM e TRAP

## SNMPv3
A ultima versão  mais recente e concentra-se principalmente em questões de segurança. Ele adiciona mecanismos **de criptografia** e **autenticação** às mensagens SNMP e não **utiliza mais strings de comunidade.** O SNMP v3 permite a **transmissão de dados totalmente criptografados** e supera as vulnerabilidades das versões anteriores.
A arquitetura SNMPv3 introduz o Modelo de Segurança Baseado no Usuário (USM) para segurança de mensagens e o Modelo de Controle de Acesso Baseado em Visualização (VACM) para controle de acesso.
O SNMPv3 suporta o identificador "Engine ID" do SNMP, que identifica exclusivamente cada entidade SNMP. Conflitos podem ocorrer se duas entidades tiverem EngineIDs duplicados. O EngineID é usado para gerar a chave para mensagens autenticadas.
Possui as mesmas operações da V1 e V2
###### USM (Modelo de segurança baseado em usuário)
Ele substitui o modelo de _community strings_ em texto claro do SNMPv1/v2c por credenciais baseadas em usuários, oferecendo autenticação criptográfica, integridade de dados e confidencialidade.
Utiliza algoritmos de hash para garantir que a mensagem se originou de um usuário valido.
Utiliza criptografia no payload do pacote SNMP utilizando algoritmos simétricos assim impedindo que seja lido caso interceptado.
Mitiga ataques onde um pacote é capturado e reenviado posteriormente, baseia-se em um identificador único do equipamento (`snmpEngineID`) e na sincronização de relógios lógicos (`EngineBoots` e `EngineTime`). Pacotes fora da janela de tempo aceitável são descartados silenciosamente.

Os algoritmos de criptografia utilizados são simetricos (DES/AES).

Possui 3 níveis de segurança, configuradas por usuários.
- **noAuthNoPriv:** Nenhuma autenticação criptográfica e nenhuma criptografia. A validação é feita apenas pelo nome de usuário correspondente. (Inseguro, similar ao SNMPv2c).
- **authNoPriv:** Autenticação habilitada (via hash), mas sem criptografia. Garante a identidade e integridade da requisição, porém o conteúdo trafega em texto claro.
- **authPriv:** Autenticação e criptografia habilitadas. O payload é cifrado e a identidade é validada. O padrão exigido em redes que trafegam dados sensíveis ou expostas à internet.

###### VACM (Modelo de Controle de Acesso Baseado em Visões)
É um mecanismo de controle de acesso adicionado na v3, é complementar ao USM (que cuida da autenticação/criptografia). Ele define oque um usuário **autenticado** pode fazer ou ver.
O USM responde "quem é você e é você mesmo?" (autenticação). O VACM responde "agora que sei quem você é, **o que você pode acessar**?". São camadas separadas e complementares.
Ele funciona cruzando algumas informações.
- Grups - As permissões são definidas por grupos. Por exemplo "Admin", "Monitoramento"...
- Security level - No USMfoi apresentado os niveis de segurança (noAuthNoPriv, authNoPriv, authPriv), então as permissões podem depender do nivel de segurança da sessão.
- views (MIB Views) - Subconjuntos da arvore MIB (OIDs). Uma view pode apenas incluir os OIDs de interface e exluir os de configuração de roteamento por exemplo.
- **access policy** - define, para cada grupo/contexto/nível de segurança, quais operações são permitidas (read, write, notify) e com qual view.
