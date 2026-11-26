# Roteiro de testes — Lethal Company Corporation

> **Ambiente simulado:** execute os testes dentro do Cisco Packet Tracer. Os exemplos abaixo usam equipamentos e endereços do projeto.

## 1. DHCP do Financeiro

Em `FIN1` (ou FN1), abra **Desktop → Command Prompt** e rode:

```text
ipconfig
ping 192.168.40.1
ping 192.168.30.2
```

**Esperado:** IP `192.168.40.x` atribuído por DHCP; gateway e servidor DHCP acessíveis.

## 2. VLAN 40 e trunk

No switch `SW-C-01`:

```text
enable
show vlan brief
show interfaces trunk
```

**Esperado:** VLAN 40 nas portas `Gi1/0/7`, `Gi1/0/8` e `Gi1/0/9`; porta `Gi1/1/1` em trunk.

No `RTR-C-01`:

```text
show ip interface brief
show running-config
```

**Esperado:** `Gi0/0/0.40`, IP `192.168.40.1`, encapsulamento dot1Q 40 e `ip helper-address 192.168.30.2`.

## 3. DNS corporativo entre prédios

No `JUR1` (Prédio B):

```text
nslookup ftp.corporation.test
```

**Esperado:** resolução para `192.168.10.3`.

## 4. FTP corporativo

Do `JUR1` (B) e do `EST1` (C):

```text
ping 192.168.10.3
ftp 192.168.10.3
```

Após autenticação com conta fictícia do laboratório, rode `dir`. **Esperado:** listar arquivos do FTP do Prédio A.

## 5. Visitantes bloqueados

No `CLIENTE-B-01` (Wi-Fi B):

```text
ipconfig
ftp 192.168.10.3
```

**Esperado:** notebook na rede `192.168.21.0/24` e FTP sem conexão. Repita a validação com o visitante do Prédio A (`192.168.11.0/24`).

## 6. Firewall Financeiro

No `FIN1` (C):

```text
ftp 192.168.1.2
```

**Esperado:** acesso e `dir` funcionando. Repita em `EST1`, `CPR1` ou `TI1`: **sem conexão**. No `FW-FIN-01`, confira a ACL com `show access-list` e a rota de retorno com `show route`.

## 7. Portal WEB entre prédios

No `ADM1` (A), abra **Desktop → Web Browser** e visite:

`http://192.168.20.3`

**Esperado:** página da Lethal Company Corporation hospedada no Prédio B.

## 8. Demonstração da redundância RIPv2

No `RTR-A-01`, veja o caminho normal:

```text
enable
show ip route
```

**Esperado:** `R 192.168.20.0/24 [120/1] via 10.0.0.2`.

Simule a falha direta A–B:

```text
configure terminal
interface Serial0/1/0
shutdown
end
show ip route
```

**Esperado após convergência:** `R 192.168.20.0/24 [120/2] via 10.0.0.9`.

Enquanto A–B estiver desativado, teste no `JUR1`:

```text
ping 192.168.10.3
ftp 192.168.10.3
```

**Esperado:** conectividade e FTP funcionais pelo caminho B → C → A.

**Restaure antes de encerrar**:

```text
configure terminal
interface Serial0/1/0
no shutdown
end
show ip route
copy running-config startup-config
```

**Esperado:** retorno à rota direta com métrica 1.

## Observações

- Os testes foram executados na simulação durante a construção do projeto.
- Concessões DHCP podem mudar endereços individuais; prefira checar o IP atual antes do ping.
- Credenciais de demonstração devem ser verificadas no próprio arquivo. Não inclua senhas reais no GitHub.
