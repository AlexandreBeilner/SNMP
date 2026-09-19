Esse é o bloco mais rico da apresentação porque é onde a teoria vira decisão de arquitetura real. Vou organizar em duas partes: **por que SNMP tem essa fama** e **o que fazer a respeito**, com exemplos concretos que você pode usar nos slides.

## Por que SNMP é considerado "lento e pesado"

A raiz do problema não é bem o protocolo em si — é **como ele costuma ser implementado nos dispositivos**:

- **SNMP roda no plano de controle, não no plano de dados.** Em roteadores, switches e OLTs, o hardware é otimizado pra encaminhar pacotes rapidamente (isso geralmente é feito em ASICs dedicados). O agente SNMP, por outro lado, roda numa CPU de gerência muito mais fraca — às vezes um processador embarcado antigo, compartilhado com SSH, interface web, logging, etc.
- **É tratado como funcionalidade secundária.** Pra muitos fabricantes, SNMP é "recurso legado obrigatório", não o caminho principal de gerenciamento. O código costuma ser menos otimizado, às vezes até single-threaded — uma consulta pesada pode travar temporariamente até a resposta do SSH/web UI do mesmo equipamento.
- **Percorrer tabelas grandes é caro.** Uma tabela de interfaces com milhares de linhas, ou uma tabela de endereços MAC, exige que o agente monte a resposta iterando a estrutura interna — isso consome CPU proporcional ao tamanho da tabela, não ao que você realmente precisa saber.

## Por que isso é ainda mais crítico em OLTs

Você tocou num ponto muito bom — vale destacar isso como um exemplo concreto na apresentação:

- OLTs geralmente têm **centenas ou milhares de ONUs/ONTs** conectadas. Uma tabela de status de ONUs via SNMP pode ser gigantesca.
- A CPU de gerência de uma OLT normalmente **não foi dimensionada pensando em polling SNMP intensivo** — o foco de engenharia do fabricante vai pro plano óptico/dados, não pra API de gerenciamento.
- Muitos fabricantes de OLT tratam SNMP como **canal secundário de monitoramento**, com o provisionamento "de verdade" acontecendo por outros protocolos proprietários (TL1, TR-069, CLI própria). Isso significa que o caminho de código do SNMP recebe menos atenção/otimização.
- É comum ver OLTs que, sob polling SNMP muito frequente ou walks completos de tabelas grandes, apresentam **picos de CPU que impactam até a estabilidade da gerência do equipamento** (não do tráfego de dados, felizmente, mas mesmo assim é um risco operacional).

Isso é uma ótima transição pro segundo bloco: já que você não pode simplesmente parar de usar SNMP nesses dispositivos (às vezes é a única opção viável), a solução é **construir a aplicação cliente de forma consciente**.

## Estratégias para não sobrecarregar o dispositivo

Organizei em categorias — você pode transformar isso numa tabela nos slides:

**1. Reduza o volume de dados por requisição**

- Prefira **GETBULK** a múltiplos GETs sequenciais, mas cuidado: `max-repetitions` muito alto pode gerar respostas fragmentadas em UDP ou sobrecarregar o agente de uma vez só. Vale testar valores conservadores (ex: começar com 10-20) por tipo de dispositivo.
- Evite fazer _walk_ de subárvores inteiras quando você só precisa de alguns OIDs específicos. Prefira `GET`/`GETNEXT` direcionados quando possível.

**2. Espalhe a carga no tempo**

- Nunca faça polling de todos os dispositivos no mesmo instante (o clássico "todo mundo às XX:00:00"). Adicione **jitter/atraso aleatório** entre o início do polling de cada equipamento.
- Ajuste o **intervalo de polling por criticidade**: métricas essenciais (status de link, por exemplo) podem ser mais frequentes; métricas menos urgentes (contadores históricos, inventário) podem rodar a cada 15-30 min ou mais.

**3. Controle concorrência**

- Limite quantas requisições simultâneas sua aplicação faz **para o mesmo dispositivo** — mesmo que seu NMS tenha capacidade de disparar 50 requisições em paralelo, o agente do outro lado pode não aguentar processar isso tudo ao mesmo tempo.
- Implemente **fila por dispositivo**, não só um pool global de conexões.

**4. Trate timeout e retry com cuidado**

- Timeout curto demais + retry agressivo cria um efeito cascata: você bombardeia um dispositivo que já está lento, piorando a situação.
- Prefira **backoff exponencial** em vez de retry imediato.

**5. Prefira eventos a polling constante quando possível**

- Use **Traps/Informs** para mudanças de estado (link up/down, por exemplo) em vez de ficar perguntando "mudou? mudou? mudou?" a cada poucos segundos.
- Isso também é um ótimo gancho pra conectar com o bloco anterior da sua apresentação (Traps e Notificações) — mostra como os tópicos se conectam na prática.

**6. Monitore a saúde do próprio dispositivo enquanto monitora**

- Ao testar/ativar polling num equipamento novo, acompanhe CPU/memória do dispositivo antes e depois. Comece conservador (intervalo maior, walks menores) e vá ajustando com dados reais, em vez de assumir que "vai aguentar".
- Documente por tipo de dispositivo qual configuração de polling é segura — isso vira conhecimento institucional valioso pra equipe.

**7. Reaproveite sessões e evite overhead extra (importante em v3)**

- Na v3, cada request tem overhead de autenticação/criptografia. Reaproveitar sessão em vez de reautenticar a cada chamada reduz carga adicional no agente.

---

Uma forma boa de fechar esse bloco na apresentação: resuma como um **princípio geral** — "SNMP em dispositivos de recursos limitados exige que a inteligência de throttling esteja no seu NMS, não no dispositivo, porque o dispositivo não vai se proteger sozinho." Isso resume bem toda a seção pra quem só vai lembrar de uma frase.

Quer que eu já monte esse conteúdo (e os outros blocos que definimos antes) direto em slides do PowerPoint?