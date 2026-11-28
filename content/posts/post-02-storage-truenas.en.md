---
title: "ZFS Storage on Recycled Hardware: Testing and Recovering Old Drives in TrueNAS SCALE (Part 2)"
date: 2026-11-11
draft: true
summary: "TrueNAS, a ZFS mirror, and a week spent discovering that three 'PASSED' drives were not to be trusted — with every command."
tags: ["homelab", "truenas", "zfs", "smart", "storage", "nfs"]
featureImage: "img/capa-post-02.png"
series: ["Zero-Cost Homelab"]
showTableOfContents: true
---

*Post 2 of the series. In [post 1]({{< ref "post-01-inventario-e-arquitetura" >}}) I covered the hardware inventory and initial architecture decisions. Here I document the storage phase, mistakes included — and there were several.*

The plan looked straightforward: install TrueNAS on an old PC, put the drives into a ZFS pool, and share the space with Proxmox over NFS. The software side went smoothly. The drives gave me a lot more to write about.

## The starting point

I had four 1TB drives for data, all with high power-on hours, plus a small SSD for the OS. Before trusting them with anything, I ran `smartctl` on each one:

```
sudo smartctl -a /dev/sdX
```

The first three results:

| Drive | Power-on hours | Reallocated sectors | Pending sectors | Uncorrectable | SMART result |
|---|---|---|---|---|---|
| Seagate ST31000528AS | 69,823 (~8 years) | **434** | **131** | **131** | PASSED |
| WD Red WD10EFRX | 66,613 (~7.6 years) | 0 | **1** | 0 | PASSED |
| WD Red WD10EFRX | 34,400 (~3.9 years) | 0 | 0 | 0 | PASSED |

All three report `PASSED`, which brings us to the first lesson: **`PASSED` only means no attribute crossed the manufacturer's threshold**, and those thresholds are very permissive. The Seagate was clearly failing, with hundreds of reallocated sectors and over a hundred pending. What actually matters are the `Reallocated_Sector_Ct`, `Current_Pending_Sector`, and `Offline_Uncorrectable` counters, which should all be zero.

> **My mistake:** on the first pass, I copied the output of the *same* drive three times. What gave it away was the serial number, identical across all three files. Always check the `Serial Number` at the top of each output.

## The long test failed on all three

`smartctl -a` shows the counters, but only a long test reads the entire drive:

```
sudo smartctl -t long /dev/sdX
sudo smartctl -l selftest /dev/sdX
```

It takes hours (roughly 2.5 to 3 per drive). Result: **all three failed**, with `Completed: read failure` and the first failing sector (the LBA) reported on each one — including the 3.9-year-old WD drive that looked completely clean.

A curious detail in the log: `LifeTime(hours)` showed 4288 and 1077 instead of 69,823 and 66,613. That is the 16-bit counter wrapping around at 65,536 hours. Not a new error, just integer truncation.

## Failing drive or something around the drive?

Three drives of different brands and ages failing at the same time points to a common cause: power supply, cables, or controller. Instead of guessing, I investigated:

1. **The counters did not change** after the tests.
2. **`dmesg` was clean:** no `hard resetting link`, no `failed command`, no `SError`. Cables and power looked innocent.
3. **Reading the failing sector directly**, using the LBA reported by the test:

```
sudo dd if=/dev/sdc of=/dev/null bs=512 skip=<LBA> count=1 iflag=direct
```

It returned `Input/output error` after nearly 3 seconds of retries. `dmesg` then logged `Unrecovered read error - auto reallocate failed`, and the drive's `Current_Pending_Sector` jumped to 1. **The issue really was on the drive platter itself.**

One warning: I accidentally used the LBA from one drive on another during one of the attempts, and `dd` "passed". Testing with the wrong address proves nothing.

## Rewriting the weak sectors

A pending sector is a sector the drive cannot read, which it only decides to reallocate when something **writes** to it. Since the drives held no data I cared about, I wrote zeros across the entire disk, forcing the drive to remap any bad sector, not just the ones I knew about:

```
sudo dd if=/dev/zero of=/dev/sdX bs=1M oflag=direct status=progress
```

⚠️ **This wipes everything.** Verify the drive by its serial number before hitting Enter.

It took about 2h30 on one drive and 2h10 on the other, at roughly 110–130 MB/s. Once finished, on both WD Red drives, `Current_Pending_Sector` dropped back to **0**, and `Reallocated_Sector_Ct` also stayed at **0**. They were "weak" sectors that became readable again after being rewritten.

The Seagate, with hundreds of reallocated and pending sectors, was not so lucky: it is a drive in active degradation, so it was **discarded**. It stayed out of the pool, and I did not even want it as a cold spare.

## The replacement drive

To get back to three drives, I used another 1TB drive I already had at home (zero cost, as always): a WD Green WD10EACS, with around 15,000 power-on hours and clean SMART stats. Two details worth noting:

- `Load_Cycle_Count` was 3114, very low. Green drives are notorious for aggressively parking their heads, and this counter shows whether the drive has already suffered from it.
- **ERC was disabled** (`smartctl -l scterc` showed `Disabled`). Without that timeout limit, the drive can spend too long trying to reread a bad sector, causing the OS or controller to drop it as dead. I capped it at 7 seconds:

```
sudo smartctl -l scterc,70,70 /dev/disk/by-id/<drive>
```

This setting resets when the drive powers down, so if you want it persistent, put the command in a startup script.

## A power outage and the controller stuck in IDE mode

Halfway through the new long tests, **the power went out at home**. The tests were aborted and I had to run them all over again. I took advantage of the downtime to fix something `dmesg` had flagged earlier: the SATA controller was running in **IDE** mode (two drives sharing a channel, `NCQ depth 0/32`). I switched it to **AHCI** in the BIOS, and the system immediately recognized the controller as `ahci`, with NCQ enabled.

After repeating the long tests, **all three drives passed** with `Completed without error`, and every error counter stayed at zero.

> **Naming lesson:** after rebooting, the `sda`, `sdb`, and `sdc` device letters swapped drives. Always use the serial number or the path under `/dev/disk/by-id/`.

## The pool decision: mirror or RAIDZ1?

| | RAIDZ1 (3 drives) | Mirror (2 drives + 1 spare) |
|---|---|---|
| Usable capacity | ~1.8 TiB | ~0.9 TiB |
| Fault tolerance | 1 drive failure | 1 drive failure |
| VM performance | Weak random IOPS | Better, especially for reads |
| Risk | A 2nd failure during resilver loses everything | Lower, and the spare drive is ready to go |

I chose a **mirror with the two WD Red drives**, keeping the WD Green as a spare. This storage will host VMs, where IOPS matter more than raw capacity, and with drives of such different ages I prefer a simpler resilver. The trade-off is ending up with roughly 0.9 TiB usable.

*(Screenshot of the pool in Storage — [insert image])*

## Datasets and NFS shares

Inside the pool (created on the `gringotts` NAS) I created a parent dataset, `hogwarts` — named after the Proxmox server that will use it — and inside it, one dataset per purpose, each with its own NFS share:

- `dormitories`: VM disks
- `restricted-section`: backups
- `library`: ISOs and templates

Two settings matter on each share: **authorized hosts** (only the Proxmox IP) and **Maproot User/Group set to `root`**, without which Proxmox — which writes as root — hits permission errors.

Having one dataset per purpose instead of a single share gives me independent snapshots, quotas, and permissions. One detail to remember: NFS does not cross ZFS dataset boundaries, so each dataset needs its own share.

## SMART tests are now cron jobs

I wanted to schedule weekly short tests and monthly long tests like always, but on TrueNAS 25.10.7 — the version I am running — **the SMART tests UI no longer exists**. According to the 25.10 release notes, scheduled SMART tests were moved to cron jobs. Here is what I set up for my weekly task (short test, Sundays at 03:00):

```
for d in /dev/disk/by-id/ata-*; do case "$d" in *-part*) continue;; esac; smartctl -t short "$d"; done
```

and the monthly one (long test, only on the WD drives, on the 10th at 02:00):

```
for d in /dev/disk/by-id/ata-WDC_*; do case "$d" in *-part*) continue;; esac; smartctl -t long "$d"; done
```

I used the `by-id` path to avoid device letter shifts, and the `case` statement to skip partitions. Cron jobs run as `root`; when I tested the commands manually without `sudo`, I got `Permission denied`.

One caveat: these cron jobs only **trigger** the tests — they will not alert me if one fails. Because of that, I check the log manually after each long test (`smartctl -l selftest`).

## What I learned (and what's next)

- A `PASSED` status proves nothing. Check the counters and run a long test.
- Verify the drive by its serial number before running any destructive command.
- Check the SATA controller mode (AHCI) and drive ERC settings.
- To recover a drive that holds no data, writing the entire disk can clear weak sectors. For important data, back it up first.
- A UPS, or at least surge/power protection, is essential: the power outage cost me hours of re-testing.
- The NAS only has **4GB of RAM**, below the recommended minimum for ZFS. That will be a limitation to benchmark later on.

In the next post: networking. I will wipe the old configuration on both Catalyst 3750 switches, connect them in a stack, and configure the management, storage, and VM VLANs. After that comes Proxmox, which will mount this storage over NFS.

---

*Hardware used: HP Compaq Pro 6300 MT (i3-3220, 4GB RAM), 2× WD Red WD10EFRX, 1× WD Green WD10EACS, 120GB Kingston SSD for the OS.*