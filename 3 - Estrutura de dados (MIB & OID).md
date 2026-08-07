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

- SMIv1 não tinha `Unsigned32` nem `Counter64` — foram adicionados no SMIv2 justamente porque redes mais rápidas estouravam contadores de 32 bits rapidamente demais.
- SMIv2 formalizou melhor a sintaxe das macros (`OBJECT-TYPE`, `MODULE-IDENTITY`, etc).