- GET, GETNEXT, GETBULK, SET, WALK
- Diferença entre WALK e GETBULK e impacto em performance de coleta
- Polling: intervalos, escala e cuidado com carga no agente

## GET, GETNEXT, GETBULK, SET, WALK
O SNMP é um protocolo request-response. O network management system (NMS) envia uma requisição e o dispositivo remoto retornar uma resposta. No entanto, como o estado de um objeto pode mudar com o tempo (por exemplo, uma interface pode estar ativa agora, mas inativa em 5 minutos), o NMS consulta o dispositivo a cada poucos minutos para obter informações atualizadas. Isso é chamado de intervalo de sondagem.

#### GET
A mensagem GET recupera um valor OID específico. O NMS especifica o OID exato e o agente responde com o valor associado a esse OID. Por exemplo, se o NMS deseja o status da interface Ge0/1, ele usa o OID correspondente e a resposta do agente contém o status da interface.

#### GETNEXT
> **Ordem lexicográfica é ordem de dicionário.** A palavra `casa` vem antes de `casaco`, que vem antes de `caso`. Você compara letra por letra, da esquerda para a direita, e a primeira diferença decide tudo. Se uma palavra acabar antes e até ali era idêntica, a mais curta vem primeiro.

