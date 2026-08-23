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
### Tabela resumo

|Aspecto|v1|v2c|v3|
|---|---|---|---|
|Formato do PDU|Trap-PDU específico (enterprise, agent-addr, generic-trap...)|Reformatado — mesmo padrão de varbinds das outras operações|Mesmo formato do v2c|
|Confirmação de entrega|Não (só trap)|Trap (sem confirmação) **e** Inform (com confirmação)|Trap e Inform, ambos disponíveis|
|Segurança|Community string em claro|Community string em claro|HMAC (auth) + AES (priv) por usuário|
|Rastreabilidade de origem|Não|Não|Sim — usuário individual|

### Informs
Informas foram adicionados na v2, ele é basicamente uma trap **com confirmação de recebimento**. Diferente da trap tradicional (fire-and-forget), o receptor precisa responder com um `GetResponse` confirmando que recebeu a mensagem — se não houver confirmação, o agente reenvia. Isso resolve o problema de traps perdidas silenciosamente por congestionamento de rede ou indisponibilidade momentânea do coletor.


## Recepção e parsing de traps
#### O trap chega como pacote UDP
 O agente (dispositivo gerenciado) envia um datagrama UDP, tipicamente para a **porta 162**, destinado ao IP do coletor/receptor (trap receiver). Diferente do polling normal (onde o gerenciador pergunta e o dispositivo responde na porta 161), aqui a iniciativa é do dispositivo.

#### O receptor (trap daemon) escuta a porta
Existe um processo dedicado escutando UDP/162 — pode ser:
- **`snmptrapd`** (parte do Net-SNMP, muito usado em Linux/ambientes open source)
- Serviços nativos de ferramentas de NMS (Zabbix trapper, PRTG SNMP Trap Receiver, SolarWinds, LibreNMS, etc.)
Esse processo precisa estar rodando com permissão para bindar a porta 162 (geralmente requer privilégio, já que é porta < 1024) e, se for v3, precisa ter as credenciais (usuário, senhas de auth/priv) configuradas previamente para conseguir decodificar o pacote.

#### Decodificação da camada SNMP (BER/ASN.1)
O SNMP usa **ASN.1** como linguagem de descrição, codificado em **BER (Basic Encoding Rules)** no wire. O receptor precisa:
1. **Decodificar o BER** para extrair a estrutura da mensagem (version, community/usuário, PDU).
2. **Validar a versão** (v1, v2c, v3) e tratar de acordo — em v3, isso inclui verificar o `msgSecurityModel`, o `EngineID`, e então autenticar (HMAC) e descriptografar (AES) o payload usando as credenciais do usuário correspondente antes de conseguir ler o conteúdo.
3. Se v1/v2c, apenas confere se a community bate com alguma configurada (sem criptografia, o conteúdo já está visível).
Se a autenticação falhar (community errada, usuário v3 não reconhecido, hash não confere), o trap é descartado, geralmente sem resposta ao remetente (silenciosamente, para não dar pista a quem está fazendo scanning).

#### Extração dos varbinds
Depois de decodificado, o PDU contém uma lista de **varbinds** (variable bindings) — pares OID + valor. Como vimos, em v2c/v3 os dois primeiros são sempre:

- `sysUpTime.0` — há quanto tempo o agente está no ar
- `snmpTrapOID.0` — o OID que identifica **qual** notificação é essa (equivalente ao "tipo de evento")

Seguidos de varbinds adicionais específicos daquele evento, por exemplo, uma trap de `linkDown` normalmente inclui o `ifIndex` e `ifDescr` da interface que caiu.

Nesse ponto, o que você tem são apenas **números** (OIDs numéricos, tipo `1.3.6.1.6.3.1.1.5.3`) e valores brutos (inteiros, strings, contadores), nada legível ainda.

#### Tradução via MIB (o passo que dá sentido a tudo)
É aqui que entra a **MIB (Management Information Base)** — um arquivo de definição (formato ASN.1/SMI) que mapeia:

```
1.3.6.1.6.3.1.1.5.3  →  linkDown
```

e descreve o tipo de dado, descrição textual e a estrutura de cada objeto. Sem carregar a MIB correspondente, o receptor mostra apenas o OID numérico cru — o que é inútil para leitura humana em ambientes com muitos fabricantes.

Processo prático:

- Você baixa/instala a MIB fornecida pelo fabricante do equipamento (Cisco, Juniper, HP, APC, etc.) no diretório de MIBs do seu trap receiver.
- O `snmptrapd` (ou equivalente) carrega essas MIBs na inicialização e faz a tradução automática: `1.3.6.1.6.3.1.1.5.3` vira `IF-MIB::linkDown`, por exemplo, e os varbinds junto ficam legíveis (`ifDescr.5 = GigabitEthernet0/1` em vez de um número de índice cru).
- Isso é conhecido como **MIB resolution** ou **OID-to-name mapping**.

#### Ação/roteamento pelo NMS
Depois de decodificado e traduzido, a trap normalmente passa por uma etapa de **regras de correlação/alerta** dentro do sistema de monitoramento:

- Filtrar por tipo de evento (ignorar `coldStart` de reinicializações programadas, por exemplo)
- Correlacionar com o inventário (a qual cliente/site/equipamento esse IP de origem pertence)
- Disparar alerta (e-mail, Slack, ticket automático) conforme severidade
- Armazenar no histórico para auditoria/dashboard

## Tratamento de enterprise-specific traps e decodificação de varbinds
Traps "genéricas" (coldStart, warmStart, linkDown, linkUp, authenticationFailure, egpNeighborLoss) são padronizadas pelo próprio protocolo SNMP e têm significado universal — qualquer NMS já sabe interpretar. Mas a grande maioria dos eventos úteis no dia a dia (temperatura alta num rack, falha de fonte redundante, disco cheio, falha de fan, evento específico de licenciamento) **não existe** no padrão genérico — são definidos pelo próprio fabricante, dentro do seu ramo particular na árvore de OIDs (`enterprise`).
Isso é o que chamamos de **enterprise-specific trap**: uma notificação cujo significado só existe na MIB proprietária daquele fabricante.

#### Como isso aparece na estrutura, por versão

**No SNMPv1** (formato antigo), a trap específica de fabricante é sinalizada explicitamente:
- `generic-trap = 6` (valor reservado que significa "enterpriseSpecific")
- `specific-trap = <código>` (o número que identifica qual evento específico é, dentro do namespace daquele fabricante)
- `enterprise = <OID>` (o OID raiz do fabricante/produto, ex: `1.3.6.1.4.1.9` para Cisco)

**No SNMPv2c/v3** (formato baseado em varbinds), não existe mais essa distinção estrutural — tudo é unificado no varbind `snmpTrapOID.0`. O OID completo já identifica diretamente o evento, esteja ele no ramo padrão (`1.3.6.1.6.3.1.1.5.x`) ou no ramo de enterprise do fabricante (`1.3.6.1.4.1.<enterprise-id>.x`). O v2c/v3 converte automaticamente uma trap v1 enterprise-specific concatenando `enterprise-OID + "0" + specific-trap` para formar o `snmpTrapOID` equivalente — isso é inclusive um ponto de atenção em ambientes com gateways/proxies traduzindo v1 para v2c.

#### O papel do enterprise number (IANA)
Todo fabricante que quer criar MIBs proprietárias registra um **Private Enterprise Number (PEN)** junto à IANA — é um número único, público, consultável em: [https://www.iana.org/assignments/enterprise-numbers](https://www.iana.org/assignments/enterprise-numbers)

Exemplos conhecidos:
- Cisco = `9`
- Microsoft = `311`
- HP = `11`
- APC = `318`

Esse número sempre aparece sob `1.3.6.1.4.1.<PEN>`, e a partir daí cada fabricante estrutura sua própria árvore como quiser, definida na MIB que ele publica.

#### Decodificação prática de varbinds, passo a passo
Vamos usar um exemplo hipotético de uma trap de temperatura alta:

**Pacote bruto recebido (simplificado):**

```
snmpTrapOID.0 = 1.3.6.1.4.1.318.0.1
varbind1: 1.3.6.1.4.1.318.1.1.1.2.2.1.0 = 1
varbind2: 1.3.6.1.4.1.318.1.1.1.2.2.2.0 = 45
```

**Passo 1 — Identificar o fabricante pelo enterprise number**  
`1.3.6.1.4.1.318` → PEN 318 → APC (American Power Conversion, nobreaks/UPS).

**Passo 2 — Carregar a MIB correspondente**  
O receptor precisa ter a MIB da APC (`PowerNet-MIB`, no caso) carregada para traduzir esses OIDs em nomes. Sem ela, você só vê os números crus.

**Passo 3 — Tradução via MIB**  
Com a MIB carregada:

```
snmpTrapOID.0        →  PowerNet-MIB::upsTrapOverload
1.3.6.1.4.1.318.1.1.1.2.2.1.0  →  upsAdvBatteryTemperature.0 = 1 (unidade: enum "abnormal")
1.3.6.1.4.1.318.1.1.1.2.2.2.0  →  upsAdvInputLineVoltage.0 = 45
```

Agora sim isso vira legível: "UPS reportou sobrecarga, com temperatura de bateria em estado anormal".

**Passo 4 — Interpretação de tipo de dado**  
Cada varbind tem um tipo ASN.1 explícito no encoding (INTEGER, OCTET STRING, Counter32, Gauge32, TimeTicks, IpAddress, OID, etc.), e a MIB frequentemente define **enums** para valores inteiros — por exemplo, `1 = normal`, `2 = warning`, `3 = critical`. Sem a MIB, o parser mostra só `2`; com a MIB, mostra `warning(2)`. Isso é frequentemente esquecido e gera falsos negativos em dashboards que só olham o número.