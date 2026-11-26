---
title: "Como começar um homelab com hardware reciclado: inventário, VLANs e nomenclatura (Parte 1)"
date: 2026-11-08
draft: false
summary: "O hardware que já tinha, as decisões de arquitetura (Proxmox, TrueNAS, ZFS, VLANs) e a regra do projeto: custo zero."
tags: ["homelab", "proxmox", "truenas", "zfs", "vlan"]
featureImage: "img/capa-post-01.png"
series: ["Homelab a Custo Zero"]
showTableOfContents: true
---

*Post 1 de uma série onde documento a construção do meu laboratório de informática — servidores virtuais, storage, rede — a partir de hardware que já tinha em casa.*

## Porquê este projeto

Trabalho em infraestruturas há mais de 20 anos, e em produção há sempre limites ao que se pode mexer, experimentar ou estragar. Este laboratório é o sítio onde posso montar, partir e voltar a montar sem consequências, e treinar com calma redes, virtualização e storage. Decidi documentá-lo em público porque explicar o que faço obriga-me a percebê-lo melhor, e porque os erros de uns poupam tempo a outros.

O objetivo é simples: montar um ambiente de virtualização e storage em casa, ir evoluindo aos poucos, e documentar publicamente cada passo — incluindo os erros e as decisões que teria feito de outra forma.

A regra do projeto é **custo zero**: não gasto nada em hardware nem em software. Tudo o que está aqui já existia em casa, e as ferramentas são gratuitas ou open-source.

## O que já tinha disponível

- **Portátil, Intel Core i7-8550U, 32GB RAM** — vai servir de hypervisor
  - Armazenamento: SSD NVMe de 500GB para o sistema
  - Rede: 1× Ethernet Gigabit (Intel I219-V) e Wi-Fi (Intel AC 8265)
- **PC antigo: HP Compaq Pro 6300 MT** — serve de storage (NAS)
  - CPU: Intel Core i3-3220 (2 núcleos, 4 threads, 3,3 GHz)
  - RAM: **4GB** DDR3 (um só módulo, com três ranhuras livres)
  - Discos de dados: 3 × 1TB — dois WD Red (WD10EFRX) e um WD Green (WD10EACS), com entre 15 mil e 66 mil horas de uso
  - Disco de sistema: SSD Kingston de 120GB
  - Havia ainda um quarto disco de 1TB (Seagate) que acabei por descartar — conto porquê mais abaixo
- **2× Switches Cisco Catalyst 3750, Gigabit, com suporte a trunk (802.1Q)**
  - Ligam-se entre si em stack (cabos de stack), e comportam-se como um único switch lógico
  - Modelo: WS-C3750G-24TS-S, com 24 portas Gigabit e 4 slots SFP, sem PoE, a correr o IOS 12.2(55)SE3 (IP Base) [confirmar que o segundo switch é igual]
  - Medidas: 45×7×29cm cada

Não é hardware topo de gama, mas esse é precisamente o ponto: mostrar o que se consegue fazer sem gastar dinheiro, e documentar as limitações reais em vez de fingir que parti de um datacenter. A mais óbvia é a memória do NAS: 4GB ficam abaixo do que a documentação do TrueNAS recomenda para ZFS, e isso vai limitar a cache de leitura. Vou medir o impacto e contá-lo mais à frente.

### Primeira lição: um "PASSED" não chega

Antes de pôr os discos num pool, corri `smartctl -a` em cada um. Os quatro diziam `PASSED`, mas o Seagate tinha 434 setores realocados e 131 pendentes. O `PASSED` só significa que nenhum atributo cruzou o limiar do fabricante, que é permissivo. O que interessa são os contadores de setores realocados, pendentes e *uncorrectable*, e um teste longo (`smartctl -t long`) que leia o disco todo. Os detalhes (e o que correu mal nos testes) ficam para o post sobre o storage.

## As decisões de arquitetura

### Hypervisor: Proxmox VE
Optei pelo Proxmox por ser gratuito, open-source, e suportar tanto VMs (KVM) como containers leves (LXC) na mesma plataforma. Vai correr no portátil.

### Storage: TrueNAS SCALE + ZFS em mirror
O PC antigo corre **TrueNAS SCALE (Community Edition)** — o TrueNAS Core foi descontinuado, com a última versão em abril de 2025, por isso o SCALE é hoje a única opção com desenvolvimento ativo.

Com três discos saudáveis, a escolha estava entre RAIDZ1 e um mirror. Fui para um **mirror com os dois WD Red**, deixando o WD Green de reserva:

- o storage vai servir VMs, e um mirror dá melhor desempenho em IOPS aleatórios do que um RAIDZ1;
- os discos têm idades muito diferentes (um deles com mais de 7 anos), e num resilver o mirror é mais simples e menos arriscado do que um RAIDZ1;
- o preço é a capacidade: cerca de 0,9 TiB úteis em vez de ~1,8 TiB.

A integração com o Proxmox vai ser via **NFS**, com um dataset por finalidade (discos das VMs, backups, ISOs), cada um com a sua partilha. É mais simples de configurar e de diagnosticar do que iSCSI, com desempenho suficiente para esta fase.

### Rede: VLANs sobre os switches Cisco
Os dois Catalyst 3750 ligam-se em stack e suportam trunk, por isso vou segmentar em pelo menos 3 VLANs, com o ID da VLAN a coincidir com o terceiro octeto do endereço:

| VLAN | ID | Rede | Propósito |
|---|---|---|---|
| Gestão | 10 | 10.0.10.0/24 | Acesso administrativo ao Proxmox, TrueNAS e switches |
| Storage | 20 | 10.0.20.0/24 | Tráfego NFS entre hypervisor e storage |
| VMs / Lab | 30 | 10.0.30.0/24 | Onde vão correr os serviços |

O portátil só tem uma porta Ethernet, por isso a gestão, o storage e as VMs vão partilhar o mesmo cabo, num **trunk 802.1Q**, com uma bridge do Proxmox preparada para VLANs. O custo é que o tráfego NFS e o das VMs competem pelo mesmo 1 Gbps — para um laboratório serve, e é um bom ponto de partida para medir o desempenho real mais à frente.

Deixo ainda reservadas, sem as configurar, as VLANs para clientes, servidores, Wi-Fi, visitantes, IoT e impressoras, para o dia em que o laboratório crescer.

![Topologia de rede do homelab](/images/infrainsight_homelab_topologia.png)

### Nomes: tudo em Hogwarts
Sou fã de Harry Potter e de mundos de fantasia em geral, por isso, em vez de acabar com `server1` e `server2`, dei aos equipamentos nomes de locais desse universo:

| Equipamento | Nome |
|---|---|
| Portátil / hypervisor Proxmox | `hogwarts` |
| PC antigo / storage TrueNAS | `gringotts` |
| Stack de switches (2× Catalyst 3750) | `diagon-alley` |
| Firewall/router (futuro) | `azkaban` |

O domínio interno será `hogwarts.lan`.

## Onde estou

- TrueNAS instalado, com um pool ZFS em mirror, datasets e partilhas NFS criadas
- Testes SMART agendados (curto semanal, longo mensal)
- Switches e VLANs: por configurar
- Proxmox: por instalar

## Próximos passos

1. Ligar os switches em stack e configurar as VLANs
2. Instalar o Proxmox no portátil
3. Ligar storage e hypervisor, validar com uma VM de teste
4. Primeiro serviço real: [Pi-hole / AdGuard Home, provavelmente]

## O que esperar dos próximos posts

Vou documentar cada fase com o processo real — incluindo o que correu mal. Os próximos posts: a história completa do storage (testes de disco que falharam, setores a recuperar, e um TrueNAS que mudou o sítio onde se agendam os testes SMART), a configuração da rede (os dois Catalyst 3750 em stack e as VLANs) e a instalação do Proxmox. O móvel de madeira onde tudo isto vai ficar montado vai sendo construído quando houver tempo, e conto-o à parte.

---

*Hardware usado neste projeto: portátil com i7-8550U e 32GB RAM, HP Compaq Pro 6300 MT com discos SATA, dois switches Cisco Catalyst 3750 em stack. Código de configuração (quando aplicável) disponível em [link do repositório Git, se aplicável].*
