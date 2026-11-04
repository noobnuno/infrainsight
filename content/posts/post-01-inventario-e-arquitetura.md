---
title: "Como comecei o meu homelab: inventário e primeiras decisões de arquitetura"
date: 2026-10-27
draft: true
tags: ["homelab", "proxmox", "truenas", "zfs", "vlan"]
---

*Post 1 de uma série onde documento a construção do meu laboratório de informática — servidores virtuais, storage, rede — a partir de hardware que já tinha em casa.*

## Porquê este projeto

[2-3 frases sobre a motivação: aprender X, ter um sítio para testar Y, construir portfólio, etc.]

O objetivo é simples: montar um ambiente de virtualização e storage em casa, ir evoluindo aos poucos, e documentar publicamente cada passo — incluindo os erros e as decisões que teria feito de outra forma.

## O que já tinha disponível

Em vez de comprar hardware novo, decidi começar com o que já tinha:

- **Portátil, 32GB RAM** — vai servir de hypervisor
  - CPU: [preencher]
  - Armazenamento local: [preencher — SSD/HDD, capacidade]
  - Placas de rede: [preencher — quantas, 1G ou 2.5G]
- **PC antigo** — serve de storage (NAS)
  - CPU/RAM: [preencher]
  - Discos de dados: 3 × 1TB — dois WD Red (WD10EFRX) e um WD Green (WD10EACS), com entre 15 mil e 66 mil horas de uso
  - Disco de sistema: SSD Kingston de 120GB
  - Havia ainda um quarto disco de 1TB (Seagate) que acabei por descartar — conto porquê mais abaixo
- **2× Switches Cisco Gigabit, com suporte a trunk (802.1Q)**
  - Modelo: [preencher, ex: Catalyst 2960]
  - Número de portas / PoE: [preencher]
  - Medidas: 45×7×29cm cada

Não é hardware topo de gama, mas é suficiente para validar a arquitetura antes de investir em algo mais dedicado — e é mais interessante documentar as limitações reais do que fingir que parti de um datacenter.

### Primeira lição: um "PASSED" não chega

Antes de pôr os discos num pool, corri `smartctl -a` em cada um. Os quatro diziam `PASSED`, mas o Seagate tinha 434 setores realocados e 131 pendentes. O `PASSED` só significa que nenhum atributo cruzou o limiar do fabricante, que é permissivo. O que interessa são os contadores de setores realocados, pendentes e *uncorrectable*, e um teste longo (`smartctl -t long`) que leia o disco todo. Os detalhes (e o que correu mal nos testes) ficam para o post sobre o storage.

## As decisões de arquitetura

### Hypervisor: Proxmox VE
Optei pelo Proxmox por ser gratuito, open-source, e suportar tanto VMs (KVM) como containers leves (LXC) na mesma plataforma. Vai correr no portátil.

### Storage: TrueNAS SCALE + ZFS em mirror
O PC antigo corre **TrueNAS SCALE (Community Edition)** — o TrueNAS Core foi descontinuado, com a última versão em abril de 2025, por isso o SCALE é hoje a única opção com desenvolvimento ativo.

Com três discos saudáveis, a escolha estava entre RAIDZ1 e um mirror. Fui para um **mirror com os dois WD Red**, deixando o WD Green de reserva:

- o storage vai servir VMs, e um mirror dá melhor desempenho em IOPS aleatórios do que um RAIDZ1;
- os discos têm idades muito diferentes (um deles com mais de 7 anos), e num resilver o mirror é mais simples e arriscado do que um RAIDZ1;
- o preço é a capacidade: cerca de 0,9 TiB úteis em vez de ~1,8 TiB.

A integração com o Proxmox vai ser via **NFS**, com um dataset por finalidade (discos das VMs, backups, ISOs), cada um com a sua partilha. É mais simples de configurar e de diagnosticar do que iSCSI, com desempenho suficiente para esta fase.

### Rede: VLANs sobre os switches Cisco
Como os switches são Gigabit e suportam trunk, vou segmentar em pelo menos 3 VLANs, com o ID da VLAN a coincidir com o segundo octeto do endereço:

| VLAN | ID | Rede | Propósito |
|---|---|---|---|
| Gestão | 10 | 10.0.10.0/24 | Acesso administrativo ao Proxmox, TrueNAS e switches |
| Storage | 20 | 10.0.20.0/24 | Tráfego NFS entre hypervisor e storage |
| VMs / Lab | 30 | 10.0.30.0/24 | Onde vão correr os serviços |

*(Diagrama da topologia de rede aqui — [inserir imagem])*

### Nomes: tudo em Hogwarts
Para não acabar com `server1`, `server2`, escolhi nomes de locais de Harry Potter:

| Equipamento | Nome |
|---|---|
| Portátil / hypervisor Proxmox | `hogwarts` |
| PC antigo / storage TrueNAS | `gringotts` |
| Switch 1 / Switch 2 | `diagon-alley` / `knockturn-alley` |
| Firewall/router (futuro) | `azkaban` |

O domínio interno será `hogwarts.lan`.

## Onde estou

- TrueNAS instalado, com um pool ZFS em mirror, datasets e partilhas NFS criadas
- Testes SMART agendados (curto semanal, longo mensal)
- Switches e VLANs: por configurar
- Proxmox: por instalar

## Próximos passos

1. Configurar as VLANs nos switches
2. Instalar o Proxmox no portátil
3. Ligar storage e hypervisor, validar com uma VM de teste
4. Primeiro serviço real: [Pi-hole / AdGuard Home, provavelmente]

## O que esperar dos próximos posts

Vou documentar cada fase com o processo real — incluindo o que correu mal. Os próximos posts: a história completa do storage (testes de disco que falharam, setores a recuperar, e um TrueNAS que mudou o sítio onde se agendam os testes SMART), a construção do móvel de madeira com rodas e prateleiras ventiladas onde tudo isto vai ficar montado, e a configuração da rede.

---

*Hardware usado neste projeto: portátil 32GB RAM, PC antigo com discos SATA, switches Cisco Gigabit. Código de configuração (quando aplicável) disponível em [link do repositório Git, se aplicável].*
