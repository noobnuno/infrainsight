---
title: "How I Started My Homelab with Recycled Hardware: Inventory, VLANs, and Naming Convention (Part 1)"
date: 2026-11-08
draft: false
summary: "The hardware I already had, the architecture decisions (Proxmox, TrueNAS, ZFS, VLANs), and the project rule: zero cost."
tags: ["homelab", "proxmox", "truenas", "zfs", "vlan"]
featureImage: "img/capa-post-01.png"
series: ["Zero-Cost Homelab"]
showTableOfContents: true
---

*Post 1 of a series where I document building my IT lab — virtual servers, storage, networking — using hardware I already had at home.*

## Why this project

I have worked in infrastructure for over 20 years, and in production there are always limits to what you can touch, experiment with, or break. This lab is the place where I can build, break, and rebuild without consequences, and practice networking, virtualization, and storage at my own pace. I decided to document it publicly because explaining what I do forces me to understand it better, and because one person's mistakes save time for others.

The goal is simple: build a virtualization and storage environment at home, evolve it step by step, and publicly document every stage — including the mistakes and the decisions I would have made differently.

The project rule is **zero cost**: I spend nothing on hardware or software. Everything here already existed at home, and the tools are free or open-source.

## What I already had available

* **Laptop, Intel Core i7-8550U, 32GB RAM** — will serve as the hypervisor

  * Storage: 500GB NVMe SSD for the OS

  * Networking: 1× Gigabit Ethernet (Intel I219-V) and Wi-Fi (Intel AC 8265)

* **Old PC: HP Compaq Pro 6300 MT** — serves as storage (NAS)

  * CPU: Intel Core i3-3220 (2 cores, 4 threads, 3.3 GHz)

  * RAM: **4GB** DDR3 (a single module, with three free slots)

  * Data drives: 3 × 1TB — two WD Red (WD10EFRX) and one WD Green (WD10EACS), with between 15,000 and 66,000 power-on hours

  * System drive: 120GB Kingston SSD

  * There was also a fourth 1TB drive (Seagate) that I ended up discarding — I explain why below

* **2× Cisco Catalyst 3750 Switches, Gigabit, with trunk support (802.1Q)**

  * Connected together in a stack (StackWise cables), behaving as a single logical switch

  * Model: both are `WS-C3750G-24TS-S`, with 24 Gigabit ports and 4 SFP slots, non-PoE, running IOS `12.2(55)SE3` (IP Base)

  * Dimensions: 45×7×29cm each

It is not top-of-the-line hardware, and that is precisely the point: showing what can be achieved without spending money, and documenting the real limitations instead of pretending I started from a datacenter. The most obvious one is the NAS memory: 4GB is below what TrueNAS documentation recommends for ZFS, and that will limit the read cache. I will measure the impact and share it later on.

### First lesson: a "PASSED" is not enough

Before putting the drives into a pool, I ran `smartctl -a` on each one. All four reported `PASSED`, but the Seagate had 434 reallocated sectors and 131 pending sectors. `PASSED` only means no attribute crossed the manufacturer's threshold, which is permissive. What matters are the reallocated, pending, and *uncorrectable* sector counters, along with a long test (`smartctl -t long`) that reads the entire drive. The details (and what went wrong during testing) are saved for the [storage post](/en/posts/post-02-storage-truenas/).

## Architecture decisions

### Hypervisor: Proxmox VE

I chose Proxmox because it is free, open-source, and supports both VMs (KVM) and lightweight containers (LXC) on the same platform. It will run on the laptop.

### Storage: TrueNAS SCALE + ZFS mirror

The old PC runs **TrueNAS SCALE (Community Edition)** — TrueNAS Core was discontinued, with its final release in April 2025, making SCALE the only option with active development today.

With three healthy drives, the choice was between RAIDZ1 and a mirror. I went with a **mirror using the two WD Red drives**, keeping the WD Green as a spare:

* the storage will serve VMs, and a mirror delivers better random IOPS performance than RAIDZ1;

* the drives are of very different ages (one of them over 7 years old), and during a resilver a mirror is simpler and less risky than RAIDZ1;

* the trade-off is capacity: roughly 0.9 TiB usable instead of ~1.8 TiB.

Integration with Proxmox will be over **NFS**, with one dataset per purpose (VM disks, backups, ISOs), each with its own share. It is simpler to configure and troubleshoot than iSCSI, with enough performance for this stage.

### Networking: VLANs over the Cisco switches

The two Catalyst 3750 switches are stacked and support trunking, so I am segmenting into 3 active VLANs in this first stage, matching the VLAN ID with the **third octet of the subnet address (`10.0.<VLAN>.0/24`)**:

| VLAN | ID | Subnet | Purpose |
| ----- | ----- | ----- | ----- |
| Management | 10 | `10.0.10.0/24` | Administrative access to Proxmox, TrueNAS, and the switch stack |
| Storage | 20 | `10.0.20.0/24` | Isolated NFS traffic between hypervisor and storage (no gateway) |
| VMs / Lab | 30 | `10.0.30.0/24` | Where services and virtual machines will run |

VLAN 20 (Storage) is kept under strict isolation, with no gateway configured and dedicated exclusively to NFS communication between the hypervisor and the NAS. In every routed network, the `.1` address is reserved for the future gateway (`azkaban`), and the default VLAN 1 is not used.

The laptop only has one Ethernet port, so management, storage, and VMs will share the same cable over an **802.1Q trunk**, using a VLAN-aware bridge in Proxmox. The trade-off is that NFS and VM traffic compete for the same 1 Gbps link — for a lab it is plenty, and it gives a solid baseline to measure real-world performance later on.

I also leave reserved, without configuring them for now, VLANs for clients (40), stable servers (50), Wi-Fi (60), guests (70), IoT (80), and printers (90), along with VLAN 999 (Parking) for disabled and unused ports, preparing the infrastructure for future expansion.

### Naming: everything in Hogwarts

I am a fan of Harry Potter and fantasy worlds in general, so instead of ending up with `server1` and `server2`, I named the hardware after locations from that universe:

| Equipment | Name |
| ----- | ----- |
| Laptop / Proxmox hypervisor | `hogwarts` |
| Old PC / TrueNAS storage | `gringotts` |
| Switch stack (2× Catalyst 3750) | `diagon-alley` (master member / stack management IP) and `knockturn-alley` (second stack member) |
| Firewall/router (future) | `azkaban` |

The internal domain will be `.hogwarts.lan`.

## Where I am now

* TrueNAS installed, with a ZFS mirror pool, datasets, and NFS shares created

* SMART tests scheduled (weekly short, monthly long)

* Switches and VLANs: to be configured

* Proxmox: to be installed

## Next steps

1. Connect the switches in a stack and configure the VLANs

2. Install Proxmox on the laptop

3. Connect storage and hypervisor, validate with a test VM

4. First real service: local DNS with ad-blocking (`owlery` — Pi-hole or AdGuard Home)

## What to expect in upcoming posts

I will document every phase with the actual process — including what went wrong. Upcoming posts: the full storage story (failed drive tests, recovering sectors, and a TrueNAS release that moved where SMART tests are scheduled), building the wooden rolling cabinet with ventilated shelves where everything will be housed, and configuring the network.

*Hardware used in this project: laptop with i7-8550U and 32GB RAM, HP Compaq Pro 6300 MT with SATA drives, two Cisco Catalyst 3750 switches in a stack.*
