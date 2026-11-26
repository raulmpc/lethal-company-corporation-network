# Arquitetura e segurança — Lethal Company Corporation

[← Voltar ao README](../README.md) · [Roteiro de testes](ROTEIRO_DE_TESTES.md)

## 1. Requisitos implementados

O laboratório representa três prédios corporativos com roteadores interligados, computadores usando DHCP, um servidor FTP corporativo, DNS, WEB, redes de visitantes em A e B e um servidor de arquivos dedicado ao Financeiro protegido por firewall.

### Distribuição dos serviços

| Prédio | LAN corporativa | Clientes adicionais | Servidores / proteção |
|---|---|---|---|
| A | `192.168.10.0/24` | Visitantes: `192.168.11.0/24` | DHCP `.10.2`; FTP `.10.3`; DNS `.10.10` |
| B | `192.168.20.0/24` | Visitantes: `192.168.21.0/24` | DHCP `.20.2`; WEB `.20.3` |
| C | `192.168.30.0/24` | Financeiro: `192.168.40.0/24` | DHCP `.30.2`; ASA outside `.30.10`; FTP protegido `192.168.1.2` |

Os IPs abreviados por coluna usam o prefixo apresentado no título da rede. Todos os departamentos mantêm computadores usando **DHCP**.

## 2. Como o tráfego entre os prédios funciona

### Exemplo: Jurídico (B) acessando FTP corporativo (A)

1. JUR1 recebe IPv4, máscara, gateway e DNS automaticamente de DHCP_B.
2. Se utilizar o nome `ftp.corporation.test`, consulta o DNS de A (`192.168.10.10`), que retorna `192.168.10.3`.
3. Para alcançar outra rede, o computador envia o tráfego ao gateway do Prédio B (`192.168.20.1`).
4. O RTR-B-01 consulta a **tabela de roteamento**. Com o enlace direto ativo, encaminha para o RTR-A-01.
5. O roteador A entrega os pacotes à rede `192.168.10.0/24`, onde está o servidor FTP.
6. As respostas voltam pelas rotas disponíveis; o FTP exige autenticação de laboratório.

O RIP propaga caminhos entre os roteadores; **não substitui** a configuração de IP, DHCP, DNS ou das ACLs.

## 3. DHCP e redes de visitantes

Cada prédio tem **um servidor DHCP com uma interface de rede**. As redes adicionais recebem concessões por meio de DHCP Relay no roteador:

- A: rede de visitantes `192.168.11.0/24` → `ip helper-address 192.168.10.2`.
- B: rede de visitantes `192.168.21.0/24` → `ip helper-address 192.168.20.2`.
- C: VLAN 40 `192.168.40.0/24` → `ip helper-address 192.168.30.2`.

DHCP fornece endereço, máscara, gateway e DNS; a escolha do pool depende da rede de origem da solicitação retransmitida.

### Restrição dos visitantes

As ACLs dos roteadores A e B foram aplicadas **na entrada das interfaces de visitantes** para impedir tráfego ao servidor FTP corporativo `192.168.10.3`. Os funcionários conservam o acesso.

## 4. Financeiro: VLAN 40 e firewall ASA 5505

**Motivação:** evitar autorizações associadas a IPs individuais que poderiam mudar com o DHCP e separar logicamente o setor dos demais departamentos.

| Elemento | Configuração |
|---|---|
| Switch C | VLAN 40 nas portas dos três PCs do Financeiro |
| Trunk SW-C-01 ↔ RTR-C-01 | IEEE 802.1Q, VLAN 40 identificada |
| Subinterface `Gi0/0/0.40` | Gateway `192.168.40.1/24`; DHCP Relay para `192.168.30.2` |
| ASA 5505 outside | `192.168.30.10/24` |
| ASA 5505 inside | `192.168.1.1/24` |
| Servidor Financeiro | `192.168.1.2/24`, gateway `192.168.1.1` |
| Rota no RTR-C-01 | `192.168.1.0/24` via `192.168.30.10` |
| Rota de retorno no ASA | `192.168.40.0/24` via `192.168.30.1` |

**Fluxo:** PC Financeiro → switch (VLAN 40) → RTR-C-01 → ASA outside → ACL → FTP-FIN-01. A ACL do ASA permite tráfego da rede `192.168.40.0/24` para o host `192.168.1.2`; a política bloqueia as demais origens conforme os testes realizados.

Separar a rede por VLAN **não é o mesmo que autenticar usuários**. A configuração cumpre o objetivo do laboratório, mas em produção seriam necessárias outras camadas de segurança.

## 5. Sub-redes dos enlaces e roteamento dinâmico

Cada ligação serial ponto a ponto utiliza um bloco **/30** (`255.255.255.252`), com dois endereços IPv4 utilizáveis:

| Enlace | Rede | Roteadores |
|---|---|---|
| A ↔ B | `10.0.0.0/30` | A: `10.0.0.1`, B: `10.0.0.2` |
| B ↔ C | `10.0.0.4/30` | B: `10.0.0.5`, C: `10.0.0.6` |
| C ↔ A | `10.0.0.8/30` | C: `10.0.0.9`, A: `10.0.0.10` |

Foram instalados módulos **NIM-2T** e utilizados cabos seriais. Nos lados DCE configurou-se o clock. Os roteadores operam com **RIPv2** e `no auto-summary`, anunciando as redes corporativas e de visitantes pertinentes.

### Por que o RIPv2 ajuda na redundância?

- Os **três enlaces** criam a possibilidade física de caminhos alternativos.
- O RIPv2 **aprende e atualiza** rotas para redes remotas.
- A métrica do RIP considera **quantidade de saltos**, por isso a ligação direta A–B tende a ser preferida.
- Com A–B desligado, o roteador A aprendeu a rede do B **via C** (`10.0.0.9`), com métrica 2 em vez de 1.
- O FTP continuou acessível no teste com JUR1. Ao reativar A–B, a rota direta foi restaurada.

> O tempo de convergência depende das condições da falha e do protocolo; não se deve afirmar que o failover é instantâneo.

## 6. Falhas investigadas durante o projeto

- **APIPA (169.254.x.x):** pools DHCP com endereçamento inicial errado e verificações do relay.
- **Subinterface VLAN 40 desaparecida:** após reinicialização de equipamento, foi necessário restaurar `Gi0/0/0.40` e salvar a configuração.
- **Notebook Wi-Fi sem encontrar SSID:** distância física ao ponto de acesso no ambiente Physical do Packet Tracer.
- **ACL de visitantes B com rede incorreta:** regra foi corrigida de `192.168.20.0/24` para `192.168.21.0/24`, na interface correta.

Esses exemplos mostram a importância de diagnosticar por camadas e confirmar o resultado dos testes antes de alterar configurações.

## 7. Demonstração

Para reproduzir as evidências, use o [roteiro completo](ROTEIRO_DE_TESTES.md). Em especial: mostrar a VLAN 40, testar um acesso FTP permitido e outro negado, consultar `show ip route` e demonstrar uma falha e a recuperação do enlace A–B.

**Nota:** toda a topologia é **simulada**. Não use as contas e senhas fictícias do arquivo `.pkt` em serviços reais.