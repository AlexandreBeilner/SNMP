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

#### Management Information Base MIB

Base de informação de gerenciamento e um database que contém coleções de informações organizadas hierarquicamente em estrutura de arvore.
Na verdade, todo o MIB é uma coleção de variáveis ​​(OIDs) armazenadas em MIBs individuais mais granulares, que formam os ramos da árvore.
Cada MIB é baseado na linguagem Abstract Syntax Notation 1 (ASN.1).
![[Pasted image 20260728173223.png|587]]


A comunicação ocorre via protocolo UDP. A porta 161 é utilizada pelo SNMP Manager para solicitar (poll) MIB dos SNMP agents. E os SNPM agents utilizam a porta 162 para enviar traps ou informações solicitadas para o SNMP Manager.
O agent escuta passivamente a porta 161 e o NMS (SNMP Manager) escuta a 162.

![[Pasted image 20260728173703.png]]

- Modelo gerenciador/agente e papel do SNMP no NMS
- Arquitetura: portas 161 (queries) e 162 (traps), transporte UDP e implicações (perda de pacote, sem garantia de entrega)
- Comparação SNMP vs. alternativas (NETCONF, gNMI, telemetria streaming) — quando cada um faz sentido

