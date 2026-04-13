---
title: "Home Server - First Configurations After Installing Proxmox"
date: 2026-04-12T15:00:00-03:00
slug: "proxmox-configuracao-inicial-home-server"
tags:
  - homelab
  - home-server
  - proxmox
  - linux
  - tutorial
draft: false
description: "What to configure right after installing Proxmox VE: storage, SMART, VLAN, backups, and ISOs - before creating the first VM."
---

Proxmox is installed, updated, and accessible through the web interface. Before creating any VMs or containers, there are some basic configurations worth doing. It is not necessary to do everything at once, but each item here prevents problems down the line.

This post describes the configurations I made on my home server right after installation. It is a general overview - each topic will be covered in a separate post.

## Recognizing disks and setting up storage

The first step is to make sure Proxmox can see all the disks and that they are available as storage.

Go to **Node → Disks** and check the list of physical disks with their model, size, and SMART status. If a disk does not show up, the issue is usually with the motherboard interface or the cable.

With the disks visible, it is time to create the storage pools. Proxmox supports ZFS natively - you can create pools directly through the web interface at **Disks → ZFS → Create ZFS**, or via terminal.

In my case, I created two pools:

- **local-zfs** → Kingston NV2 1TB (nvme0n1) - Proxmox installation, VMs, and ISOs
- **zpool** → 2x WD Red Plus 4TB (sda + sdb) - data, backups, and media

The WD pool is a ZFS mirror: both drives are mirrored, so the usable capacity is ~4TB - not 8TB. 

> The advantage is redundancy. If one drive fails, no data is lost. I will cover the mirror creation in detail in the next post.

After creating the pools, they appear under **Datacenter → Storage**. Verify that the content types are correctly configured for each pool - ISO images, VM disks, VZDump backup file.

## Checking disk SMART status

SMART monitors disk health and is the first line of defense against hardware failure. In Proxmox, it is found under **Node → Disks** - the **Health** column shows the status of each disk.

What you want to see is **PASSED** on all of them. If **FAILED** or **Unknown** appears, investigate before moving on.

For details via SSH:

```bash
# Full report for a disk
smartctl -a /dev/sda

# Short test (~2 minutes)
smartctl -t short /dev/sda

# Test results
smartctl -l selftest /dev/sda
```

Pay attention to the **Reallocated_Sector_Ct**, **Current_Pending_Sector**, and **Offline_Uncorrectable** fields, which indicate potential disk problems. Any value other than zero in these three deserves attention and may indicate the disk needs to be replaced. 

> **New drives should be at zero.**

## Making the bridge VLAN-aware

By default, Proxmox creates a `vmbr0` bridge that connects all VMs to the same network. This works, but if you have multiple NICs or want to segment traffic by service, enabling VLAN-awareness on the bridge is worth it.

With VLAN-aware enabled, you can assign a different VLAN tag to each VM - separating management traffic from media traffic, for example.

To enable it: **Node → Network → vmbr0 → Edit → check VLAN Aware → OK → Apply Configuration**.

In my setup, the N5105 has 4x 2.5G NICs. The `vmbr0` uses `enp1s0` as the uplink to the router. The other three are available for future use - whether as dedicated uplinks for specific VMs or for isolated storage network traffic.

If you have a managed switch with 802.3ad support, it is also possible to aggregate two NICs into a LACP bond to double the available bandwidth. The configuration involves creating a `bond0` with the desired interfaces and using it as the bridge port instead of the direct NIC.

## Setting up automatic backups

Before creating any VM, configure the backup schedule. It is much easier to do this now than after services are already running.

Go to **Datacenter → Backup → Add**:

| Field | Recommended value |
|---|---|
| Node | Your node (e.g., `pve`) |
| Storage | Where to save (e.g., data pool) |
| Day of week | Days that make sense |
| Start Time | Low-usage time (e.g., 03:00) |
| Compression | ZSTD |
| Mode | Snapshot |

**Snapshot** mode is ideal - it performs the backup without stopping the VM. **Stop** mode is more reliable for data consistency, but it interrupts the service during the process.

With storage configured and the schedule set, Proxmox handles the rest automatically.

## Uploading ISOs

To create VMs, you need the operating system ISOs available in storage. Proxmox does not download ISOs automatically - you need to upload them or provide a URL.

Go to **Storage → local (or whichever pool you designated for ISOs) → ISO Images → Upload** or **Download from URL**.

ISOs worth having before you start:

- **Debian 12** - solid base for Linux services, well tested with Proxmox
- **Ubuntu Server 24.04 LTS** - good compatibility, extensive documentation
- **VirtIO drivers** - required if you plan to create Windows VMs. Direct download at [fedorapeople.org/groups/virt/virtio-win/](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/)

VirtIO drivers are necessary because Windows does not natively recognize the paravirtualized adapters from QEMU/KVM. Without them, the Windows VM will not see the disk or network during installation.

## IOMMU and PCI Passthrough

IOMMU is the technology that allows passing a physical hardware device directly into a VM - as if the device were physically connected to it, with no software intermediary.

The most common use case is GPU passthrough for Windows VMs, or passing a storage controller directly into a NAS VM.

To enable it on Proxmox with Intel:

```bash
# Edit GRUB parameters
nano /etc/default/grub

# Change the GRUB_CMDLINE_LINUX_DEFAULT line to:
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

# Apply
update-grub
```

Then add the VFIO modules to `/etc/modules`:

```
vfio
vfio_iommu_type1
vfio_pci
```

Reboot and verify:

```bash
dmesg | grep -e DMAR -e IOMMU
```

If lines with `IOMMU enabled` are returned, it is active.

**Note for Intel N-series CPUs (N5105, N100, N200):** these CPUs have VT-d support on paper, but behavior varies depending on the motherboard. The Topton N5105 has coarse-grained IOMMU groups - devices that should be in separate groups end up sharing one, which limits passthrough. Run the tests, but do not rely on this as a primary feature of the setup.

---

With all this in place, the environment is properly prepared: disks organized, health monitored, network configured, backups scheduled, and ISOs available. The next step is a detailed configuration of the ZFS Mirror with the WD Red Plus drives - pool creation, scheduled scrub, and automatic snapshots.