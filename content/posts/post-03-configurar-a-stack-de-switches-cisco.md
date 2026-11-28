---
title: "Homelab com Cisco Catalyst 3750G: stack, VLANs e routing inter-VLAN sem complicações (Parte 3)"
date: 2026-11-20
draft: true
summary: "Configuração do stack Cisco Catalyst 3750G do zero: limpeza de configuração no master, eleição de prioridade, VLANs 10, 20, 30 e 999, trunks 802.1Q e hardening de acesso via SSHv2."
tags: ["homelab", "cisco", "networking", "vlan", "switching", "ssh"]
featureImage: "img/capa-post-03.png"
series: ["Homelab a Custo Zero"]
showTableOfContents: true
---

*Post 3 da série sobre a construção do meu laboratório de informática. Neste post configuro os dois switches Cisco que vão ligar tudo o resto: o portátil (hypervisor), o PC de storage e, mais tarde, o router/firewall.*

## Ponto de partida

Os dois switches são **Cisco Catalyst 3750** (modelo WS-C3750G-24TS-S), com 24 portas Gigabit e 4 SFP, sem PoE, a correr IOS 12.2(55)SE3 com imagem IP Base. Ambos vieram com configuração antiga de outra rede, por isso o primeiro passo foi limpar tudo.

Como vão ficar ligados por cabos de stack, funcionam como um único switch lógico. Chamei-lhe **diagon-alley**, seguindo a nomenclatura do homelab.

## 1. Limpar a configuração

Num switch com configuração antiga, é preciso apagar três coisas: a configuração de arranque, a base de dados de VLANs e a configuração corrente, que só é apagada com o reload.

```text
write erase
delete flash:vlan.dat
reload
```

Ao fazer `reload` aparece a pergunta *"System configuration has been modified. Save?"*. Responde **no**, para não gravar a configuração antiga de volta.

**Erro que me aconteceu:** fiz `write erase` num dos membros e o outro manteve a configuração. Na stack, o master é quem guarda a configuração corrente e a sincroniza para os restantes membros. Por isso, para limpar a stack, o correto é fazer tudo a partir do master.

## 2. Definir o master da stack

O master é escolhido pela prioridade. Por defeito, as duas unidades têm prioridade 1, e o escolhido pode mudar em cada reinício. Para o fixar:

```text
configure terminal
switch 2 priority 15
switch 1 priority 1
end
write memory
```

Confirma com `show switch`. O membro com prioridade 15 deve aparecer como `Master`.

## 3. VLANs e IPs

Segui o plano de endereçamento que definimos para o homelab: uma VLAN por função e o terceiro octeto do IP igual ao ID da VLAN (`10.0.<VLAN>.0/24`).

| VLAN | Nome | Rede | Função |
|---|---|---|---|
| 10 | GESTAO | 10.0.10.0/24 | Acesso administrativo |
| 20 | STORAGE | 10.0.20.0/24 | Tráfego NFS entre Proxmox e TrueNAS |
| 30 | LAB | 10.0.30.0/24 | VMs e serviços de teste |
| 999 | PARKING | — | Portas não usadas |

```text
configure terminal
hostname diagon-alley
ip domain-name hogwarts.lan
vtp mode transparent
!
vlan 10
 name GESTAO
vlan 20
 name STORAGE
vlan 30
 name LAB
vlan 999
 name PARKING
!
interface Vlan10
 description Gestao diagon-alley
 ip address 10.0.10.2 255.255.255.0
 no shutdown
```

Usei `vtp mode transparent` para que os switches não propaguem VLANs para outros equipamentos. Como as VLANs são criadas localmente, não há razão para as deixar sujeitas a VTP.

## 4. Trunks para o hypervisor e o storage

As portas que ligam ao portátil e ao PC de storage transportam as três VLANs etiquetadas (802.1Q), por isso ficam em modo trunk.

```text
interface GigabitEthernet1/0/1
 description TRUNK hogwarts (Proxmox)
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 spanning-tree portfast trunk

interface GigabitEthernet1/0/2
 description TRUNK gringotts (TrueNAS)
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 spanning-tree portfast trunk
```

**Erro que me aconteceu:** o comando `switchport mode trunk` foi rejeitado com a mensagem *"trunk encapsulation is Auto"*. No 3750, ao contrário de switches mais recentes, o encapsulamento não é automático e tem de ser definido antes do modo trunk.

## 5. Portas não usadas e segurança de acesso

As portas que não vou usar ficam desligadas, na VLAN de parking, com proteção contra BPDU. Assim, se alguém ligar um cabo por engano, não consegue alterar a topologia (as portas 25 a 28 correspondem aos quatro uplinks SFP de cada switch).

```text
interface range GigabitEthernet1/0/3 - 28, GigabitEthernet2/0/1 - 28
 description NAO USADA
 switchport mode access
 switchport access vlan 999
 spanning-tree portfast
 spanning-tree bpduguard enable
 shutdown
```

Para o acesso ao switch, desativei o HTTP e deixei apenas SSH versão 2, com passwords cifradas:

```text
service password-encryption
enable secret <password-enable>
username admin privilege 15 secret <password-admin>
no ip http server
ip ssh version 2
crypto key generate rsa modulus 2048
line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0
```

## Verificação

Como ainda não há equipamentos ligados, o `show interfaces trunk` aparece vazio. Isto é normal: o comando só lista portas ativas. Por isso, verifiquei o estado com os comandos que funcionam sem links:

```text
show switch
show vlan brief
show interfaces description
show running-config interface GigabitEthernet1/0/1
show ip ssh
```

## O que vem a seguir

O switch está pronto para receber os equipamentos. No próximo post vou ligar o portátil e configurar o Proxmox com a bridge VLAN-aware, e depois o TrueNAS no PC de storage. Aí vamos ver se o trunk funciona de facto.

---

*Equipamento usado: 2× Cisco Catalyst 3750 (WS-C3750G-24TS-S) em stack, IOS 12.2(55)SE3 IP Base.*