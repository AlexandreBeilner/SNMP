
Instalar o **Net-SNMP**, que contém tanto:
- **`snmpd`** — o agente (daemon que responde às requisições, simulando um dispositivo)
- **`snmpget`, `snmpwalk`, `snmpbulkwalk`, `snmpset`, `snmptrap`** — as ferramentas de cliente (o "manager")

```
sudo apt install snmpd snmp snmp-mibs-downloader
```

Editar `/etc/snmp/snmpd.conf`

```
sudo nano snmpd.conf
```

Adicionar algo como isso ao final do arquivo
```
# ===== Comunidades (SNMPv1/v2c) =====
# rocommunity <string> <origem> [OID restrito, opcional]
rocommunity public-teste-pdi  127.0.0.1

# rwcommunity <string> <origem> [OID restrito, opcional]
rwcommunity private 127.0.0.1

# ===== Onde o agente escuta =====
agentAddress udp:127.0.0.1:161

# ===== Log (opcional, ajuda a debugar) =====
# nível de log e destino
# %d = data no nome do arquivo de log
```

Precisa ser comentada a linha que ja vem pro padrão `agentAddress`. Senão da erro na inicialização.  E pode ser comentado as linhas de `rocommunity` pré existentes para que a nossa regra de community string funcione.
### Explicando cada diretiva

| Diretiva                        | O que faz                                                                                                                                                                |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `rocommunity <string> <origem>` | Define uma community de **somente leitura** (Read-Only). `<origem>` restringe quem pode usar essa community — `127.0.0.1` significa só o próprio localhost.              |
| `rwcommunity <string> <origem>` | Define uma community de **leitura e escrita** (Read-Write) — necessária para o `snmpset` funcionar.                                                                      |
| `syslocation`                   | Valor que será retornado no OID `sysLocation` (1.3.6.1.2.1.1.6.0).                                                                                                       |
| `syscontact`                    | Valor retornado no OID `sysContact` (1.3.6.1.2.1.1.4.0).                                                                                                                 |
| `agentAddress`                  | Endereço/porta onde o `snmpd` escuta. Formato: `udp:IP:porta`. Se quiser escutar em todas interfaces, use `udp:161` (mas para testes locais, `127.0.0.1` é mais seguro). |
#### RESTRINGINDO ACESSO POR OID
Fazendo uma community string enchergar apenas uma parte da arvore MIB.
```
rocommunity public 127.0.0.1 -V somente-system
view somente-system included .1.3.6.1.2.1.1
```
Isso faz com que a community `public` só consiga fazer GET/Walk dentro do ramo `system` (1.3.6.1.2.1.1), útil para entender como "views" funcionam em ambientes corporativos, onde nem tudo é exposto para qualquer community.

### SNMPv3 (autenticação e criptografia)
SNMPv3 não usa `rocommunity`/`rwcommunity`, usuários são criados via um arquivo separado, `snmpd.conf` só referencia grupos/views. O jeito mais simples de criar um usuário v3 é:
```
sudo systemctl stop snmpd
sudo net-snmp-create-v3-user -ro -A "senhaAutenticacao" -X "senhaPrivacidade" -a SHA -x AES usuario_teste
sudo systemctl start snmpd
```

Para testar 
```
snmpget -v3 -u usuario_teste -l authPriv -a SHA -A "senhaAutenticacao" -x AES -X "senhaPrivacidade" 127.0.0.1 1.3.6.1.2.1.1.5.0
```

### TESTANDO COMANDO SNMP

## GET
```
snmpget -v2c -c public-teste-pdi 127.0.0.1 1.3.6.1.2.1.1.1.0
```
Como esse comando é comproto então:
`snmpget` : operação que esta sendo realizada
`-v2c` : verção utilizada
`-c public-teste-pdi` : a community string criada dentro do arquivo de condifuração
`127.0.0.1` : destino
`OID`: 1.3.6.1.2.1.1.1.0

Isso nos retornar
```
iso.3.6.1.2.1.1.1.0 = STRING: "Linux ixcsoft-VJFE69F11X-B0221H 6.14.0-37-generic #37~24.04.1-Ubuntu SMP PREEMPT_DYNAMIC Thu Nov 20 10:25:38 UTC 2 x86_64"
```

## GETNEXT
```
snmpgetnext -v2c -c public-teste-pdi 127.0.0.1 1.3.6.1.2.1.1
```
Aqui mandamos o OID `1.3.6.1.2.1.1`, e a resposta foi:
```
iso.3.6.1.2.1.1.1.0 = STRING: "Linux ixcsoft-VJFE69F11X-B0221H 6.14.0-37-generic #37~24.04.1-Ubuntu SMP PREEMPT_DYNAMIC Thu Nov 20 10:25:38 UTC 2 x86_64"
```
Ou seja, retornou a mesma coisa que antes, mas com um comando diferente e passando um OID diferente. 

## WALK
```
snmpwalk -v2c -c public-teste-pdi 127.0.0.1 1.3.6.1.2.1.1
```

Aqui o nosso retorno foi
```
iso.3.6.1.2.1.1.1.0 = STRING: "Linux ixcsoft-VJFE69F11X-B0221H 6.14.0-37-generic #37~24.04.1-Ubuntu SMP PREEMPT_DYNAMIC Thu Nov 20 10:25:38 UTC 2 x86_64"
iso.3.6.1.2.1.1.2.0 = OID: iso.3.6.1.4.1.8072.3.2.10
iso.3.6.1.2.1.1.3.0 = Timeticks: (139506) 0:23:15.06
iso.3.6.1.2.1.1.4.0 = STRING: "Me alexandre.beilner@ixcsoft.com.br"
iso.3.6.1.2.1.1.5.0 = STRING: "ixcsoft-VJFE69F11X-B0221H"
iso.3.6.1.2.1.1.6.0 = STRING: "Sitting on the Dock of the Bay"
iso.3.6.1.2.1.1.7.0 = INTEGER: 72
iso.3.6.1.2.1.1.8.0 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.2.1 = OID: iso.3.6.1.6.3.10.3.1.1
iso.3.6.1.2.1.1.9.1.2.2 = OID: iso.3.6.1.6.3.11.3.1.1
iso.3.6.1.2.1.1.9.1.2.3 = OID: iso.3.6.1.6.3.15.2.1.1
iso.3.6.1.2.1.1.9.1.2.4 = OID: iso.3.6.1.6.3.1
iso.3.6.1.2.1.1.9.1.2.5 = OID: iso.3.6.1.6.3.16.2.2.1
iso.3.6.1.2.1.1.9.1.2.6 = OID: iso.3.6.1.2.1.49
iso.3.6.1.2.1.1.9.1.2.7 = OID: iso.3.6.1.2.1.50
iso.3.6.1.2.1.1.9.1.2.8 = OID: iso.3.6.1.2.1.4
iso.3.6.1.2.1.1.9.1.2.9 = OID: iso.3.6.1.6.3.13.3.1.3
iso.3.6.1.2.1.1.9.1.2.10 = OID: iso.3.6.1.2.1.92
iso.3.6.1.2.1.1.9.1.3.1 = STRING: "The SNMP Management Architecture MIB."
iso.3.6.1.2.1.1.9.1.3.2 = STRING: "The MIB for Message Processing and Dispatching."
iso.3.6.1.2.1.1.9.1.3.3 = STRING: "The management information definitions for the SNMP User-based Security Model."
iso.3.6.1.2.1.1.9.1.3.4 = STRING: "The MIB module for SNMPv2 entities"
iso.3.6.1.2.1.1.9.1.3.5 = STRING: "View-based Access Control Model for SNMP."
iso.3.6.1.2.1.1.9.1.3.6 = STRING: "The MIB module for managing TCP implementations"
iso.3.6.1.2.1.1.9.1.3.7 = STRING: "The MIB module for managing UDP implementations"
iso.3.6.1.2.1.1.9.1.3.8 = STRING: "The MIB module for managing IP and ICMP implementations"
iso.3.6.1.2.1.1.9.1.3.9 = STRING: "The MIB modules for managing SNMP Notification, plus filtering."
iso.3.6.1.2.1.1.9.1.3.10 = STRING: "The MIB module for logging SNMP Notifications."
iso.3.6.1.2.1.1.9.1.4.1 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.2 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.3 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.4 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.5 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.6 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.7 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.8 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.9 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.10 = Timeticks: (0) 0:00:00.00
```

O OID utilizado foi como base foi `1.3.6.1.2.1.1`.  Podemos analisar esse retorno da seguinte forma.
