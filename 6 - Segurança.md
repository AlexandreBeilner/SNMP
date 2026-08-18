- Hardening de community strings e migração para v3
- Restrição de acesso por ACL/firewall nas portas SNMP
- Boas práticas em ambiente multi-cliente

## Hardening de community strings e migração para v3
Como ja falado em partes anteriores do estudo, O SNMP nas versões 1 e 2c usa **community strings** como único mecanismo de "autenticação"

#### Hardening de community strings
Quando falamos de **Hardening de community strings** estamos querondo fazer a v1 e v2 ficarem mais seguras mesmo utilizando community strings. Podemos fazer algumas coisas quanto a isso.
- **Trocar os defaults imediatamente** — nunca `public`/`private`. Use strings longas, aleatórias, sem relação com nome da empresa/dispositivo.
- **Separar RO e RW** — strings diferentes para read-only e read-write; idealmente desabilitar RW se não for estritamente necessário.
- **Restringir por ACL/source IP** — configurar o dispositivo para aceitar SNMP apenas de IPs específicos (servidores de monitoramento conhecidos).
- **Isolar em VLAN de gerência** — tráfego SNMP não deve trafegar em [[VLAN]] de usuário; use uma rede de management dedicada e segmentada.
- **Bloquear a porta 161/162 UDP no perímetro** — nunca expor SNMP à internet.
- **Rotação periódica** das strings, com processo documentado.
- **Desabilitar SNMP write** (`private`) em dispositivos onde não é usado — é o vetor mais crítico, pois permite reconfiguração remota.
- **Logging/SNMP trap monitoring** para detectar tentativas de acesso não autorizado.

#### Migração para a V3
Como falado anteriormente no estudo o v3 resolve o problema na raiz, adicionando mecanismos de segurança eficientes, como o USM e o VACM. O passo a passo para a migração seria mais ou menos esse:
**1. Inventariar todos os dispositivos SNMP-capable e verificar suporte a v3**  
Levante uma lista completa de tudo que hoje responde a SNMP (switches, roteadores, firewalls, nobreaks, storage, impressoras, etc.) e confirme na documentação de cada modelo se ele suporta v3. Equipamentos muito antigos ou com firmware desatualizado podem só suportar v1/v2c — nesse caso, entram na lista de exceções que vão precisar de compensação via hardening (VLAN de gerência, ACL) até serem substituídos ou atualizados.

**2. Criar usuários SNMPv3 individuais (não compartilhados) por operador/sistema**  
Diferente da community string única, o v3 trabalha com contas de usuário nomeadas. Em vez de todo mundo usar a mesma "senha", cada operador ou sistema de monitoramento (Zabbix, um analista específico, etc.) recebe seu próprio usuário, com seu próprio algoritmo de autenticação (SHA-256 ou superior) e de criptografia (AES-128+), além de senhas próprias de autenticação e de privacidade. Isso permite depois saber exatamente quem acessou o quê.

**3. Configurar EngineID único por dispositivo**  
O EngineID é um identificador que o protocolo v3 usa internamente para gerar as chaves de autenticação/criptografia de cada usuário naquele dispositivo específico. Se você clonar a configuração de um equipamento para outro (imagem, template, backup restaurado) sem trocar o EngineID, os dois ficam com o mesmo identificador, o que enfraquece a segurança e pode permitir ataques de replay entre eles. Por isso, sempre confirme que cada dispositivo tem o seu próprio EngineID, gerado ou regenerado após qualquer clonagem.

**4. Testar em paralelo, mantendo v2c ativo apenas durante a validação**  
Na prática, não dá pra simplesmente desligar v2c e ligar v3 de uma vez em produção sem risco de perder visibilidade de monitoramento. O ideal é configurar v3 nos dispositivos e nas ferramentas de monitoramento, testar se tudo está coletando normalmente (traps, polling, etc.), e só depois desligar v2c. O ponto de atenção aqui é: esse período de "convivência" deve ser curto e monitorado, não virar uma configuração permanente por comodidade.

**5. Desabilitar v1/v2c completamente após validação**  
Esse é o passo que mais costuma ser esquecido. Depois que v3 está funcionando e validado, é preciso _de fato_ remover as configurações antigas de community string do dispositivo. Deixar v2c ativo "só por garantia" anula todo o ganho de segurança da migração, porque o atacante sempre vai tentar o caminho mais fraco disponível — se v2c ainda responde, ele vai usá-lo.

**6. Integrar com AAA/TACACS+ ou RADIUS quando a plataforma suportar**  
Em ambientes maiores, gerenciar usuários SNMPv3 manualmente em dezenas ou centenas de dispositivos vira um problema operacional (troca de senha, remoção de usuário que saiu da empresa, etc.). Quando o fabricante suporta, integrar a autenticação com um servidor centralizado (TACACS+ ou RADIUS) facilita esse ciclo de vida e reduz a chance de credenciais esquecidas ou desatualizadas.

**7. Atualizar as ferramentas de monitoramento para autenticar via v3**  
Não adianta configurar os dispositivos e esquecer da ferramenta que faz o polling (Zabbix, PRTG, LibreNMS, SolarWinds, etc.). É preciso atualizar os templates/hosts nessas ferramentas com as credenciais v3 corretas (usuário, protocolo de auth, protocolo de priv, senhas), senão o monitoramento simplesmente para de funcionar no momento em que v2c for desligado.

**8. Documentar e versionar as configurações de segurança aplicadas**  
Registre o que foi feito: quais dispositivos migraram, quais usuários foram criados, quais algoritmos foram padronizados, data da migração e quem executou. Isso é essencial tanto para auditoria quanto para facilitar a vida de quem for dar manutenção ou investigar um incidente depois — sem essa documentação, cada troca de equipe vira um processo de "redescobrir" como o ambiente está configurado.

## Restrição de acesso por ACL/firewall nas portas SNMP
Mesmo com uma community string forte (ou já usando v3), o serviço SNMP continua sendo um serviço de rede escutando em uma porta (por padrão, **UDP 161** para consultas/polling e **UDP 162** para traps/notificações). Se qualquer host da rede consegue enviar pacotes para essa porta, você depende 100% da autenticação do protocolo para se proteger — e é exatamente essa segunda camada de defesa que a restrição por ACL/firewall adiciona: **limitar quem sequer consegue tentar falar com o serviço**, independente da credencial usada.
Existem alguns locais onde aplicar essas restrições:

**1. No próprio dispositivo (ACL local)** 
A maioria dos equipamentos de rede (Cisco, Juniper, MikroTik, etc.) permite configurar uma ACL diretamente vinculada ao processo SNMP, restringindo quais IPs de origem podem consultar aquele agente.
Aqui, mesmo que alguém descubra a community string, o dispositivo simplesmente ignora a requisição se ela não vier de um dos IPs permitidos.

**2. No firewall de perímetro/segmentação**
Além da ACL local, o firewall que separa a VLAN de gerência de outras redes deve ter regras explícitas permitindo tráfego SNMP **apenas** entre os hosts de monitoramento e os dispositivos gerenciados — bloqueando qualquer outra origem por padrão (deny-all implícito). Isso é o que realmente impede um atacante posicionado em outra parte da rede de sequer alcançar a porta 161/162.

**3. No perímetro com a internet**
Regra básica e não-negociável: **SNMP nunca deve estar exposto à internet**. Bloquear UDP 161/162 de entrada em qualquer firewall de borda. Existem inclusive campanhas de scanning massivo na internet especificamente procurando por SNMP com community `public` respondendo publicamente — é um dos primeiros vetores testados por ferramentas automatizadas de reconhecimento (Shodan, Masscan, etc.).

##### Boas práticas específicas
- **Allowlist, não blocklist**: permita explicitamente os poucos IPs que precisam de acesso (servidores NMS, estações de administradores) e negue tudo o resto por padrão — nunca o contrário.
- **Restringir também a origem dos traps**: se os dispositivos enviam traps (UDP 162) para um coletor central, o firewall deve permitir esse tráfego apenas na direção e nos hosts esperados, evitando que a porta fique aberta para receber de qualquer origem.
- **Aplicar em ambas as direções**: tanto a ACL controlando quem pode fazer _polling_ no dispositivo, quanto a regra controlando para onde o dispositivo pode enviar _traps_.
- **Revisar periodicamente**: listas de IPs autorizados tendem a ficar desatualizadas (servidor de monitoramento antigo que não existe mais, IP que mudou). Trate isso como parte da revisão periódica de firewall, não como configuração "configurar uma vez e esquecer".
- **Logging de tentativas negadas**: manter log das tentativas bloqueadas (como no exemplo acima com `deny any log`) ajuda a identificar reconhecimento/varredura acontecendo contra a infraestrutura antes de um ataque mais sério.

## Boas práticas em ambiente multi-cliente
Esse cenário é comum em provedores de serviço, MSPs (Managed Service Providers), datacenters compartilhados ou qualquer ambiente onde **um único time de infraestrutura gerencia equipamentos ou monitora dispositivos de múltiplos clientes distintos**. Aqui o risco central muda de foco: não é só "alguém de fora acessar o SNMP", é **um cliente conseguir enxergar dados ou credenciais de outro cliente**, ou o próprio provedor perder controle sobre quem tem acesso a quê.
##### Principais riscos específicos desse cenário
- **Vazamento cruzado de credenciais**: se a mesma community string ou usuário v3 for reaproveitado entre clientes, um vazamento em um ambiente compromete todos.
- **Segmentação insuficiente**: tráfego SNMP de clientes diferentes trafegando na mesma VLAN/rede de gerência sem isolamento.
- **Falta de rastreabilidade**: sem usuários individuais por cliente/operador, fica impossível saber depois "quem" acessou o equipamento de qual cliente.
- **Escopo de acesso mal definido**: um operador ou sistema de monitoramento com acesso amplo demais, podendo consultar/alterar equipamentos fora do escopo do cliente que ele deveria atender.

##### Boas praticas recomendadas
- **Credenciais únicas por cliente (nunca compartilhadas)**  
Cada cliente/tenant deve ter suas próprias credenciais SNMPv3 (usuário, senha de auth, senha de priv) — nunca uma community string ou usuário genérico reutilizado entre ambientes diferentes. Isso limita o "blast radius" caso uma credencial vaze.
- **Segmentação de rede por cliente**  
Idealmente, cada cliente tem sua própria VLAN de gerência (ou, em ambientes maiores, VRF/rede lógica isolada), evitando que tráfego SNMP de clientes diferentes compartilhe o mesmo segmento de broadcast. Isso também reduz o risco de vazamento em caso de sniffing.
- **ACLs específicas por cliente/dispositivo**  
As regras de firewall/ACL de acesso SNMP devem ser granulares: o servidor de monitoramento do Cliente A só pode falar com os IPs de gerência dos equipamentos do Cliente A — nunca com equipamentos de outros clientes, mesmo que estejam na mesma infraestrutura física do provedor.
- **Auditoria e logging centralizados, mas segregados por cliente**  
Mantenha logs de acesso SNMP (quem consultou o quê, quando) de forma centralizada para facilitar auditoria interna, mas com capacidade de filtrar/segregar por cliente — importante inclusive para eventuais exigências contratuais de compliance (LGPD, contratos de SLA com cláusulas de segurança, etc.).
