---
title: "Home Server - Why I Built One and the Hardware I Chose"
date: 2026-03-02T15:00:00-03:00
slug: "home-server-por-que-montar-e-hardware-escolhido"
tags:
  - homelab
  - home-server
  - proxmox
  - hardware
draft: false
---

I built a Home Server. Not out of luxury or hobbyism: it was because the alternatives I was using were costing me too much time. The idea had been sitting in my head for months - years, really. Until a few problems started irritating me enough to finally do something about it.

## The problem with streaming

The first was streaming. Not that I'm against paying for a service - the problem is the unpredictability. Titles "disappear" from the catalog without notice. Anime I wanted to rewatch suddenly no longer available. Old TV shows that never made it to any platform. Niche films that exist in one catalog today and "vanish" tomorrow.

> **I want control over what I watch, when I watch it and on which device. A personal media server solves that.**

The second reason was cloud storage - *I'll talk about that in another post* - and also for study. I work in technology and have always wanted a local environment to test things, spin up services, experiment with configurations without depending on a cloud provider or the machine I use daily.

With those goals in mind, I started researching.

## The options I found

The research was lengthy: forums, videos, Reddit, homelab communities. The options that came up most often were TrueNAS, UnRAID, CasaOS and OpenMediaVault. Each with a different focus: TrueNAS is very solid for storage and ZFS, UnRAID is popular but paid, CasaOS has a nice interface and is more geared toward beginners, OMV is open-source and good for a simple NAS.

Pre-built hardware solutions also came up: Synology and ASUSTOR. They come with everything included, hardware and software designed for plug-and-play use. For a lot of people it's the best choice. But it wasn't what I wanted.

I wanted to choose the parts. I wanted to understand what was running underneath. I wanted to be able to expand, swap out, configure things the way that made sense for my use case. Off-the-shelf solutions take away exactly the part of the process that interests me.

## Why Proxmox

That's when I landed on **Proxmox VE**. It's a Type 1 hypervisor - meaning it runs directly on the hardware with no operating system underneath - based on Debian Linux. Free, open-source, with a complete web interface and used in real enterprise environments.

The concept is simple: instead of one physical server per service, you virtualize everything. Each application runs in an isolated VM or container (*explained below*), sharing the same hardware. Proxmox manages this through a web interface, without needing a keyboard and monitor permanently connected to the machine.

Beyond the practical use, it's a lab for studying virtualization, networking, containers and storage.

```
A VM (Virtual Machine) is a computer "simulated" by software - it has its own operating system,
dedicated CPU and memory, as if it were a separate machine existing inside the physical one.

A container is lighter: it doesn't simulate the entire hardware, it only isolates the application
from the rest of the system. It shares the host OS kernel and uses far less space and memory.
The trade-off is that you lose the full isolation a VM provides.
```

Decision made. Now it was time to choose the hardware.

## The hardware I chose

### Topton N5105 Motherboard

The Topton N5105 comes with an **Intel Celeron N5105** soldered onto the board: four cores, four threads, TDP of **10W**. That number was what caught my eye right away.

> **A server that stays on 24/7 has to be energy-efficient.**

Beyond the low power draw, what convinced me was the I/O package:

- 4x 2.5G network ports with Intel i226-V B3, essential for separating management, VM and storage traffic on distinct interfaces
- 6x SATA III and 2x M.2 - plenty of room to grow in storage
- 2 SODIMM DDR4 slots with support up to 32GB
- Native Wake-on-LAN and PXE

It's an industrial mini PC board, built to run non-stop. No noisy cooler, no parts that can't handle continuous use.

### Kingston NV2 1TB SSD (M.2 NVMe)

The system drive: where Proxmox lives, along with ISOs, containers and VMs. The Kingston NV2 achieves **3500 MB/s** read and **2100 MB/s** write via PCIe. For a homelab, more than enough.

1TB gives comfortable headroom before needing to rethink storage organization.

### MOVESPEED 1TB SSD (NVMe PCIe 4.0 x4)

A second M.2, dedicated to projects and tests that need their own disk. Right now it's running a **Bitcoin node**, which continuously reads and writes a considerable amount of data. Having a separate disk for that isolates the workload and keeps it from competing with the main system.

### 2x WD Red Plus 4TB in ZFS Mirror

For data, movies, series and backups, I chose two **WD Red Plus 4TB** drives in a **ZFS Mirror**.

The WD Red Plus is built for NAS: runs 24/7, 5400 RPM (less heat and less noise than 7200 RPM) and has firmware optimized for continuous operation. The Mirror choice came from a framework I found during research:

> **Yes, extensive benchmark testing had been done on this - well done, too.**

```
Do you care about your data?
No:   go with striped.
Yes, continue:

How many disks do you have?
1:      ZFS is not for you.
2:      Mirror
3-5:    RAIDZ1
6-10:   RAIDZ1 x 2
10-15:  RAIDZ1 x 3
16-20:  RAIDZ1 x 4
```

With two disks, Mirror is the way. You get 4TB usable (the second disk works entirely as a copy) and if one fails, you lose nothing. ZFS also checksums every block, so silent data corruption is detected before it becomes a problem.

### 2x Corsair Vengeance 16GB DDR4 2666MHz - 32GB total

SODIMM, which is what the board requires. Two 16GB sticks in dual channel, totaling **32GB**.

For anyone planning to virtualize, RAM is the most critical resource. Every VM needs allocated memory. With 32GB I have room to run multiple services simultaneously without the system starting to complain.

### Cooler Master G500 Gold 500W PSU

*Yes, 500W for a board with a 10W TDP sounds like overkill. But there's a reason.*

An **80 Plus Gold** PSU operates above 90% efficiency through most of its load curve. Cheap PSUs without certification lose more energy converting AC to DC - what the community calls **power vampirism**: watts that leave the outlet but never reach the components. With the server running 24/7, that waste shows up on the electricity bill month after month.

On top of that, a PSU sized with headroom runs far from its limit, meaning less heat and longer lifespan. And there's margin left over to add drives or any other expansion without needing to replace the PSU.

### Wisecase CK-16 Case

This is where a touch of reuse comes in. The case is a **Wisecase CK-16** - my first gaming PC case, bought many years ago. Leaving it sitting idle didn't make sense. It ended up being a better choice than it first appeared: 4x 5.25" bays, 4 internal 3.5" bays and 2 exposed - **plenty of space for the WD Red Plus drives and any future expansion**. Modern cases typically come with one or two HDD bays at most. The CK-16 handles that without buying anything new.

---

## Hardware Summary

| Component | Model | Notes | Unit price | Total (w/ shipping) |
|---|---|---|---|---|
| Motherboard / CPU | Topton N5105 | TDP 10W, 4x 2.5G NIC | R$ 627.16 | R$ 627.16 |
| System SSD | Kingston NV2 1TB NVMe | Proxmox, VMs, ISOs | R$ 258.99 | R$ 276.31 |
| Project SSD | MOVESPEED PCIe 4.0 | Bitcoin node and tests | R$ 236.15 | R$ 236.15 |
| Data HDDs | 2x WD Red Plus 4TB | ZFS Mirror | R$ 444.99 | R$ 889.98 |
| Memory | 2x Corsair Vengeance 16GB DDR4 | 32GB dual channel | R$ 209.99 | R$ 426.33 |
| PSU | Cooler Master G500 Gold 500W | 80 Plus Gold | R$ 399.90 | R$ 399.90 |
| Case | Wisecase CK-16 | Reused | - | - |
| SATA 6-pin Splitter Cable | - | Installation accessory | R$ 49.05 | R$ 49.05 |
| **Total** | | | | **R$ 2,904.88** |

---

With the hardware defined, the next step was installing Proxmox. That's coming in the next post.
