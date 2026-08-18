- Traps (v1/v2c) vs. Informs (com confirmação)
- Recepção e parsing de traps — aplicação direta no que você vem fazendo em Go (`gosnmp/gosnmp`)
- Simulação e teste com `snmptrap`
- Tratamento de enterprise-specific traps e decodificação de varbinds

## Traps (v1/v2c) vs. Informs (com confirmação)
### TRAP V1
- Usa um formato de PDU próprio, diferente das outras operações (`Trap-PDU`, tipo específico, não reaproveita a estrutura de `GetResponse`).
- Contém campos específicos como `enterprise` (OID identificando o tipo de dispositivo/fabricante), `agent-addr` (endereço do agente que gerou a trap), `generic-trap` (tipo genérico: coldStart, warmStart, linkDown, linkUp, authenticationFailure, etc.), `specific-trap` (código específico do fabricante) e `time-stamp`.
- Sem confirmação de recebimento — é **fire-and-forget** (enviado via UDP e esquecido, não há garantia de entrega).
- Segurança = community string em texto claro, mesma fragilidade discutida antes.

### TRAP SNMPv2c
Na v2 do SNMP a trap foi reestruturada. O v2c abandonou o formato específico do v1 e passou a usar a mesma estrutura de PDU das outras operações (baseada em varbinds — pares OID/valor), incluindo obrigatoriamente `sysUpTime.0` e `snmpTrapOID.0` como os dois primeiros varbinds da mensagem. Isso simplificou a implementação e tornou o formato mais consistente com o resto do protocolo.

### TRAP V3
A v3 não muda novamente o PDU da trap, mas muda a camada de segurança em cima da operação.
- Traps na v3 podem (e devem) usar **authPriv** — ou seja, a notificação trafega autenticada (HMAC) e criptografada (AES), da mesma forma que uma consulta normal.
- Isso resolve um ponto que muita gente esquece: mesmo migrando o _polling_ (Get/GetNext) para v3, é comum encontrar ambientes onde as traps continuam sendo enviadas em v2c/v1 "porque sempre funcionou assim" — deixando esse canal como o elo fraco remanescente.

### Informs
Informas foram adicionados na v2, ele é basicamente uma trap **com confirmação de recebimento**. Diferente da trap tradicional (fire-and-forget), o receptor precisa responder com um `GetResponse` confirmando que recebeu a mensagem — se não houver confirmação, o agente reenvia. Isso resolve o problema de traps perdidas silenciosamente por congestionamento de rede ou indisponibilidade momentânea do coletor.
