
Oque é o SNMP?

SNMP (Simple Network Management Protocol) - É um protocolo para gerenciamento de dispositivos de rede. 

Ele possui alguns componentes: 
NMS (Network Manager System) - É o ponto central da comunicação, ele que vai solicitar e receber dados.

Agents - Agents são os dispositivos gerenciados pelo NMS. Podem ser roteadores, switches, OLTs, etc...

A comunicação ocorre via protocolo UDP, nas portas 161 e 162.
No Agente a porta critica é a 161, ele fica escutando essa porta, e nela que ele recebe as requisições.
No NMS a porta critica é a 162, é nessa porta que ele fica escutando e esperando os envios do agente.
**Como funciona na prática quando o NMS inicia a comunicação**
1. O **NMS** monta o pacote GET-REQUEST.
2. O sistema operacional do NMS escolhe uma **porta de origem efêmera** (ex: 54231) para esse pacote.
3. O pacote sai do NMS assim:
    - **Origem:** IP do NMS : porta 54231
    - **Destino:** IP do Agente : porta 161
4. O Agente recebe esse pacote na porta 161. Ele **lê o cabeçalho UDP** do pacote recebido e vê de onde veio (IP do NMS, porta 54231).
5. O Agente monta o GET-RESPONSE e envia de volta **invertendo origem/destino**:
    - **Origem:** IP do Agente : porta 161
    - **Destino:** IP do NMS : porta 54231 _(a mesma porta efêmera que ele leu no pacote recebido)_
6. O NMS, que ficou "escutando" nessa porta efêmera 54231 (só pra essa transação), recebe a resposta ali.

**Como funciona o fluxo do Inform Quando o Agente inicia a comunicação**
1. O **Agente** monta o INFORM-REQUEST e escolhe uma **porta efêmera de origem** (ex: 45678).
2. Envia:
    - **Origem:** IP do Agente : porta 45678
    - **Destino:** IP do NMS : porta **162**
3. O **NMS** recebe na porta 162, lê o cabeçalho UDP e vê de onde veio (IP do Agente, porta 45678).
4. O NMS processa o Inform e manda de volta um **GET-RESPONSE** (a confirmação):
    - **Origem:** IP do NMS : porta 162
    - **Destino:** IP do Agente : porta 45678 _(a porta efêmera que ele leu no pacote recebido)_
5. O Agente recebe essa confirmação na porta efêmera 45678, casa com o **Request-ID** que ele mesmo gerou, e sabe que o Inform chegou com sucesso.

Exitem 3 principais versões, v1, v2c e v3.

Na v1 nos temos as seguintes funcionalidades:
GET, GETNEXT, SET e TRAPS. Ela é pouco segura pois utiliza community strings para autenticação. Essas strings são textos puros transitando entre a rede. 

A v2c possui tuddo que  a v1 tem, mas teve algumas adições:
GETBULK e INFORMES. Ela também é pouco segura pelo mesmo motivo da v1

A v3 por sua vez possui tudo que v2c, mas adiciona uma coisa importante
Uma camada de segurança robusta, agora os dados trafegados entre a rede são criptografados, e os Agentes que utilizam a v3 podem ter um sistema interno de usuários com permissões que limita recursos e oque pode ser visto e feito.

Alguns termos que são importantes de conhecer antes de seguir

OID - Object Identifier - É um valor no formato 1.2.3.4.1.1.2.3, ele representa um valor do dispositivo, podendo ser a temperatura, a potencia, etc...

MIB (Management information base) - É o banco de dados que armazena os OIDs. É como se fosse o indice de um livro. Aqui cada OID vai apontar para uma informação especifica. 
Existem MIBs padrões que seguem RFCs, e MIBs proprietárias, que não seguem nenhum padrão, quem decide é o dono dela. 

Vamos entender os Métodos agora

GET - O GET vai buscar no agente um valor com base em um OID.

GETNEXT - O NMS envia um OID, o Agente vai responder com o próximo valor com base nesse OID.

SET - Utilizado para setar valores no Agente.

TRAPS - É quando o Agente configurado para que quando algo aconteça envia uma mensagem para o NMS, sem que ele tenha solicitado.

GETBULK - É passado uma lista de OIDs que vão ser buscados todos de uma vez, diminuindo a quantidade de indas e vindas.

INFORMER - Semelhante ao TRAP, mas aqui ele espera uma confirmação que o NMS recebeu a informação. Isso é importante pois a comunicação é UDP, logo não é orientada a conexão.



