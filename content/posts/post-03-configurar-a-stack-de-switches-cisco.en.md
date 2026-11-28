---
title: "Homelab with Cisco Catalyst 3750G: Stack, VLANs, and Hassle-Free Inter-VLAN Routing (Part 3)"
date: 2026-11-20
draft: true
summary: "Configuring the Cisco Catalyst 3750G stack from scratch: wiping config on the master, priority election, VLANs 10, 20, 30, and 999, 802.1Q trunks, and SSHv2 access hardening."
tags: ["homelab", "cisco", "networking", "vlan", "switching", "ssh"]
featureImage: "img/capa-post-03.png"
series: ["Zero-Cost Homelab"]
showTableOfContents: true
---

*Post 3 in the series about building my IT lab. In this post I configure the two Cisco switches that will connect everything else: the laptop (hypervisor), the storage PC, and, later on, the router/firewall.*

## Starting point

The two switches are **Cisco Catalyst 3750** units (model WS-C3750G-24TS-S), with 24 Gigabit ports and 4 SFP slots, non-PoE, running IOS 12.2(55)SE3 with the IP Base image. Both came with old configurations from another network, so the first step was wiping everything.

Since they will be connected via stack cables, they operate as a single logical switch. I named it **diagon-alley**, following the homelab naming convention.

## 1. Wiping the configuration

On a switch with an old configuration, you need to clear three things: the startup configuration, the VLAN database, and the running configuration, which is only cleared on reload.

```text
write erase
delete flash:vlan.dat
reload
```

When running `reload`, you will see *"System configuration has been modified. Save?"*. Answer **no**, so the old configuration is not written back.

**Mistake I made:** I ran `write erase` on one member and the other kept its configuration. In a stack, the master holds the running configuration and syncs it to the other members. So to wipe the stack, the right way is to run everything from the master.

## 2. Setting the stack master

The master is elected by priority. By default, both units have priority 1, and the elected master can change on each reboot. To lock it in:

```text
configure terminal
switch 2 priority 15
switch 1 priority 1
end
write memory
```

Verify with `show switch`. The member with priority 15 should show up as `Master`.

## 3. VLANs and IPs

I followed the addressing plan defined for the homelab: one VLAN per function, with the third IP octet matching the VLAN ID (`10.0.<VLAN>.0/24`).

| VLAN | Name | Subnet | Purpose |
|---|---|---|---|
| 10 | GESTAO | 10.0.10.0/24 | Administrative access |
| 20 | STORAGE | 10.0.20.0/24 | NFS traffic between Proxmox and TrueNAS |
| 30 | LAB | 10.0.30.0/24 | VMs and test services |
| 999 | PARKING | — | Unused ports |

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

I used `vtp mode transparent` so the switches do not propagate VLANs to other devices. Since VLANs are created locally, there is no reason to leave them subject to VTP.

## 4. Trunks for the hypervisor and storage

The ports connecting to the laptop and the storage PC carry all three tagged VLANs (802.1Q), so they are set to trunk mode.

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

**Mistake I made:** `switchport mode trunk` was rejected with *"trunk encapsulation is Auto"*. On the 3750, unlike newer switches, encapsulation is not automatic and must be set before trunk mode.

## 5. Unused ports and access security

Unused ports are shut down, placed in the parking VLAN, and protected against BPDUs. That way, if someone plugs in a cable by mistake, they cannot alter the topology (ports 25 to 28 correspond to the four SFP uplinks on each switch).

```text
interface range GigabitEthernet1/0/3 - 28, GigabitEthernet2/0/1 - 28
 description NAO USADA
 switchport mode access
 switchport access vlan 999
 spanning-tree portfast
 spanning-tree bpduguard enable
 shutdown
```

For switch access, I disabled HTTP and allowed only SSH version 2, with encrypted passwords:

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

## Verification

Since no devices are connected yet, `show interfaces trunk` appears empty. That is expected: the command only lists active trunk ports. Instead, I checked the state with commands that work without active links:

```text
show switch
show vlan brief
show interfaces description
show running-config interface GigabitEthernet1/0/1
show ip ssh
```

## What's next

The switch is ready for the hardware. In the next post I will connect the laptop and configure Proxmox with a VLAN-aware bridge, followed by TrueNAS on the storage PC. That is when we will see the trunk working in practice.

---

*Hardware used: 2× Cisco Catalyst 3750 (WS-C3750G-24TS-S) stacked, IOS 12.2(55)SE3 IP Base.*