---
title: "Home Server - Proxmox VE Installation"
date: 2026-04-01T15:00:00-03:00
slug: "proxmox-instalacao-home-server"
tags:
  - homelab
  - home-server
  - proxmox
  - linux
  - virtualization
  - tutorial
draft: false
description: "A complete step-by-step guide to installing Proxmox VE 8 on a home server: creating a bootable USB, running the installation wizard, configuring networking, setting up repositories, and first commands."
---

In the [previous post](../../../03/02/home-server-por-que-montar-e-hardware-escolhido/) I covered the reasons behind building a home server and detailed every piece of hardware I chose. With the hardware assembled and on the bench, the next step was getting the system up and running. That's where Proxmox comes in.

This post covers the installation from scratch: from writing the ISO to a USB drive all the way to a fully updated system ready to host its first VMs and containers.

---

## What is Proxmox VE

**Proxmox Virtual Environment** is a Type 1 hypervisor - meaning it runs directly on the hardware, without a host operating system underneath. It is based on Debian Linux, is free and open-source, and ships with a full web management interface.

The core premise is straightforward: instead of dedicating one physical machine per service, you virtualize everything. Each application runs in an isolated VM or container, sharing the same physical hardware. Proxmox orchestrates all of this through a browser-accessible interface, so you don't need a keyboard and monitor permanently connected to the server.

### Type 1 vs Type 2 hypervisors

There are two categories of hypervisors:

- **Type 1 (bare metal):** runs directly on hardware. Proxmox, VMware ESXi, and Microsoft Hyper-V are examples. Better performance, lower overhead.
- **Type 2 (hosted):** runs on top of a conventional operating system. VirtualBox and VMware Workstation fall here. Easier to install, but with more layers between the hardware and the VM.

For a server that will run 24/7 and host real services, Type 1 is the right path.

### VMs vs Containers in Proxmox

Proxmox supports two virtualization models:

```
VM (KVM/QEMU)
├── Complete, independent operating system
├── Dedicated (or shared with limits) CPU and memory
├── Can run any OS: Windows, Linux, BSD
└── Full isolation from the host

Container (LXC)
├── Shares the host kernel
├── Linux only
├── Much lighter on memory and CPU
└── Near-instant startup
```

The choice between them depends on the workload. Windows requires a VM. Lightweight Linux services - Pi-hole, Nginx Proxy Manager, Uptime Kuma - run perfectly in LXC containers with a fraction of the resources a full VM would consume.

### Why Proxmox and not the alternatives

The previous post already mentioned the alternatives I evaluated. For full context:

- **VMware ESXi:** became paid after the Broadcom acquisition. Out of the question for a homelab.
- **Microsoft Hyper-V:** tied to the Windows Server ecosystem. Not what I wanted.
- **TrueNAS Scale:** excellent for NAS, but its storage focus didn't fit what I needed as a primary platform.
- **Proxmox:** free, open-source, used in enterprise production, native ZFS support, full web interface. Easy decision.

---

## Hardware used in this installation

Just to provide context on what was used and serve as a reference for anyone with a similar setup:

| Component | Model |
|---|---|
| Motherboard / CPU | Topton N5105 - 4 cores, 10W TDP |
| System SSD | Kingston NV2 1TB NVMe (`/dev/nvme0n1`) |
| Projects SSD | MOVESPEED 1TB PCIe 4.0 (`/dev/nvme1n1`) |
| Data HDDs | 2x WD Red Plus 4TB (`/dev/sda`, `/dev/sdb`) |
| Memory | 2x Corsair Vengeance 16GB DDR4 - 32GB total |
| PSU | Cooler Master G500 Gold 500W |
| NICs | 4x Intel i226-V 2.5G (integrated on the Topton board) |

---

## What you'll need

Before starting, gather:

- A **USB drive** of at least 8 GB (its contents will be erased)
- An **Ethernet cable** connected to your router - the installation configures the network and you'll need access right after
- A **monitor and keyboard** temporarily connected to the server for the installation process
- A separate computer to create the bootable USB and, afterward, to access the web interface

---

## Part 1 - Bootable USB drive

### Downloading the ISO

Go to **proxmox.com/downloads**, click **Proxmox Virtual Environment**, and download the latest ISO. The file name will follow the format `proxmox-ve_8.x-x.iso`.

> Always download the latest available version. The installation process described here follows the Proxmox VE 8 wizard, which is what's running on the server.

### Writing with Balena Etcher

**Balena Etcher** is the most straightforward tool for this. It's free, runs on Windows, macOS, and Linux, and automatically validates the write. Download it at **balena.io/etcher**.

With Etcher open:

1. **Flash from file** → select the `.iso` file you downloaded
2. **Select target** → select the USB drive. Double-check which one it is if you have more than one connected
3. **Flash!** → wait. The process takes between 2 and 5 minutes and Etcher validates the write at the end

> If you're on Windows, it's normal for the system to show a message asking to format the USB drive after writing - it doesn't recognize the Linux filesystem. **Do not format it.** Dismiss the dialog and the drive is ready.

**Alternatives to Etcher:** Rufus (Windows), Ventoy, or `dd` on Linux/macOS.

```bash
# Write with dd on Linux (replace /dev/sdX with your drive)
dd if=proxmox-ve_8.x-x.iso of=/dev/sdX bs=4M status=progress && sync
```

---

## Part 2 - Installation

### Configuring boot order in BIOS

Plug the USB drive into the server, power it on, and enter the BIOS (usually `DEL`, `F2`, or `F10` - depends on the motherboard; on the Topton N5105 it's `DEL`).

What to adjust:

1. **Virtualization enabled** - look for `Intel VT-x` or `Virtualization Technology`. It must be `Enabled`. Without this, Proxmox cannot create VMs.
2. **Boot order** - place the USB first. Alternatively, use the Boot Menu (`F11` or `F12`) for a one-time boot from the USB without permanently changing the order.
3. **Boot Mode** - leave it on **UEFI**. Proxmox supports UEFI and it's the recommended mode.

Save with `F10` and the server will reboot, loading from the USB drive.

### Installer initial screen

When it loads, the Proxmox installer menu appears. Select **Install Proxmox VE (Graphical)** and wait for the wizard's graphical environment to load.

### Screen 1 - EULA

Read the terms and click **I agree** to continue.

### Screen 2 - Target disk

Here you choose where Proxmox will be installed and the filesystem.

**Target Harddisk:** select the destination disk. In my case, the **Kingston NV2 1TB** (`/dev/nvme0n1`).

Click **Options** to open the filesystem options.

**Chosen filesystem: ZFS (RAID0)**

Yes, ZFS on a single disk. The reason is that Proxmox benefits significantly from ZFS features even without RAID: data integrity checksums on all blocks, native snapshots (useful for rolling back the system before a problematic update), and transparent compression.

```
ext4    → simple, reliable, no overhead
ZFS     → checksums, snapshots, compression - more features, more RAM
```

With 32GB of RAM in the machine, ZFS memory overhead on a single disk is negligible. The choice was worth it.

Leave the other ZFS options at their defaults (`ashift=12`, compression `on`) and click **Next**.

### Screen 3 - Location and keyboard

- **Country:** Brazil
- **Timezone:** America/Sao_Paulo
- **Keyboard Layout:** Brazilian Portuguese (or `br-abnt2`, depending on your keyboard)

Click **Next**.

### Screen 4 - Password and email

**Password:** set the password for the `root` user. This password will be used for both the web interface login and SSH access.

> Use a strong password. Write it down somewhere safe - a password manager such as **Bitwarden**, a piece of paper in a locked drawer, whatever works. Losing root access to a server without a KVM is an unnecessary headache.

**Email:** an address for system notifications. Proxmox sends alerts about storage issues, failed backups, and available updates here.

Click **Next**.

### Screen 5 - Network configuration

This is the most important screen of the installation. What you set here determines how you'll access the server afterward.

**Management Interface:** select the network adapter for management. The Topton N5105 has four Intel i226-V NICs, which appear as `enp4s0`, `enp5s0`, `enp6s0`, and `enp7s0`. I used `enp4s0`, the first port.

The settings I configured:

```
Hostname (FQDN): berserk.local
IP Address:      192.168.1.10/24
Gateway:         192.168.1.1
DNS Server:      1.1.1.1
```

Adapt the IP and gateway to your network's range. The key points are that the IP must be static and within your router's subnet.

> Write down this information before clicking Next. You'll need the IP to access the web interface right after installation.

Click **Next**.

### Screen 6 - Confirmation

The wizard displays a summary of everything. Verify:

- Correct disk selected
- Correct filesystem (ZFS in my case)
- Correct IP and hostname
- Password written down

Click **Install**.

### Installation process

The installer formats the disk, installs the Debian base, installs Proxmox VE on top of it, and configures the bootloader. On an NVMe SSD this takes between 2 and 3 minutes.

When it finishes:

```
Installation successful!
Please remove the installation medium and press Enter to reboot.
```

Remove the USB drive and press Enter. The server will reboot, loading Proxmox from the SSD.

---

## Part 3 - First access to the web interface

After booting, the server displays an access address on the terminal:

```
Please use your web browser to configure this server - connect to:
https://192.168.1.10:8006/
```

On your computer's browser, navigate to:

```
https://192.168.1.10:8006
```

### Certificate warning

The browser will display a security warning - something like "Your connection is not private." This is expected: Proxmox uses a self-signed TLS certificate by default, and browsers don't recognize it as trusted.

Click **Advanced** → **Proceed to 192.168.1.10**. On a local network with a server you control, this is safe.

### Login

```
Username: root
Password: [password set during installation]
Realm:    Linux PAM standard authentication
```

Click **Login**.

### "No valid subscription" pop-up

On the first login - and on every subsequent login until you have a paid subscription - Proxmox displays this warning. It informs you that you're using the version without commercial support.

The feature set is identical between the free and paid versions. The subscription only covers professional technical support from Proxmox GmbH. Click **OK** and move on.

---

## Part 4 - Initial configuration

### Repositories: disable Enterprise, enable No-Subscription

By default, Proxmox points to the `enterprise.proxmox.com` repository, which requires an active subscription. Without one, any attempt to update the system returns an authentication error.

The fix is to disable the Enterprise repository and enable No-Subscription, which is the free repository carrying the same package versions.

Navigate to:

```
Datacenter → berserk → Updates → Repositories
```
> **Replace berserk with your own node name.**

What to do:

1. Find the line with `enterprise.proxmox.com` - click it, then click **Disable**
2. Click **Add** → select **No-Subscription** → **Add**

Expected result:

```
https://enterprise.proxmox.com/debian/pve   → Disabled
http://download.proxmox.com/debian/pve      → Enabled (No-Subscription)
```

> There is a third repository called `pvetest`, containing packages in testing. **Do not enable it** in an environment you want stable. No-Subscription covers everything you need.

### Update the system

With the correct repositories configured, go to:

```
Datacenter → berserk → Updates
```
> **Replace berserk with your own node name.**

Click **Refresh** to update the package list. Wait for it to finish.

Then click **Upgrade**. A terminal opens listing the packages to be updated. Confirm with `Y` + Enter and wait for the download and installation to complete.

```bash
# The same can be done via SSH
apt update && apt dist-upgrade -y
```

After finishing, reboot the server if the upgrade includes a new kernel (the terminal will advise you):

```bash
reboot
```

---

## Part 5 - Dashboard and what to monitor

After logging in, go to:

```
Datacenter → berserk → Summary
```
> **Replace berserk with your own node name.**

The dashboard shows the current state of the server. Mine shows:

```
CPU:     4x Intel Celeron N5105 @ 2.00GHz
RAM:     9.17 GiB used / 31.19 GiB total
HD:      69.18 GiB used / 93.93 GiB (ZFS system volume)
Kernel:  Linux 6.8.12-20-pve
Version: pve-manager/8.4.17
```

### Storage pools

Under **Datacenter → berserk → Disks → ZFS** and **Storage**, you'll see the volumes the installer created and any you add later. My setup looks like this:

| Pool | Type | Use |
|---|---|---|
| `local (berserk)` | Directory | ISOs, backups, container templates |
| `local-lvm (berserk)` | LVM-Thin | Disk images, containers |
| `local-zfs (berserk)` | ZFS | Disk images, containers (Kingston NV2) |
| `zpool (berserk)` | ZFS | Disk images, containers (WD Red Plus Mirror) |

The `zpool` is the ZFS Mirror built from the two WD Red Plus 4TB drives - mirrored data lives here.

### Network interfaces

Under **Datacenter → berserk → Network**:

```
enp4s0  → Physical management NIC
enp5s0  → Physical NIC (available for VMs)
enp6s0  → Physical NIC (available for VMs)
enp7s0  → Physical NIC (available for VMs)
vmbr0   → Linux Bridge on top of enp4s0 (management + VMs)
```

The `vmbr0` bridge is what allows VMs to communicate with the physical network. When creating a VM, you'll assign `vmbr0` as its network interface.

---

## Useful commands via SSH

With Proxmox running, you can manage most things from the terminal over SSH:

```bash
# Connect to the server
ssh root@192.168.1.10

# Proxmox version
pveversion

# Status of main services
systemctl status pve-cluster pveproxy pvedaemon

# Update the system
apt update && apt dist-upgrade -y

# List VMs
qm list

# List LXC containers
pct list

# ZFS pool status
zpool status

# Verify data integrity (ZFS scrub)
zpool scrub zpool

# Disk space by pool
zfs list
```

---

## Troubleshooting - When things don't go as expected

### Web interface won't open

If `https://192.168.1.10:8006` doesn't load:

```bash
# Check if the service is running
systemctl status pveproxy

# Restart the service
systemctl restart pveproxy

# Confirm the port is listening
ss -tlnp | grep 8006
```

Also verify that the IP you configured during installation is correct. If you entered the wrong IP in the wizard, edit the network configuration file:

```bash
nano /etc/network/interfaces
# Adjust the address under iface vmbr0
systemctl restart networking
```

### Update error - Enterprise repository

```
E: Failed to fetch https://enterprise.proxmox.com/...
   401 Unauthorized
```

Cause: Enterprise repository active without a subscription. Follow the steps in Part 4 to disable Enterprise and enable No-Subscription.

Via terminal this also works:

```bash
# Disable Enterprise repo
sed -i 's/^deb/# deb/' /etc/apt/sources.list.d/pve-enterprise.list

# Add No-Subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

apt update
```

### VM won't start

Check the most common causes:

- Available RAM: if all VMs combined exceed physical memory, Proxmox will refuse to start more
- Storage space: the VM's target disk needs free space
- Virtualization enabled in BIOS: if Intel VT-x isn't active, KVM VMs won't work

```bash
# View error logs for a specific VM (replace 100 with the VM ID)
journalctl -u pve-cluster | tail -50
qm showcmd 100
```

### VM has no network access

Verify that the `vmbr0` bridge is correctly configured:

```bash
ip link show vmbr0
bridge link show
```

In the web interface, confirm that the VM has a network adapter assigned and that it points to `vmbr0`. The recommended driver is **VirtIO** - better performance than the emulated `e1000`.

---

## Next steps

Proxmox is installed, updated, and has the disks recognized. What comes next in the series:

- **ZFS Mirror setup** with the WD Red Plus drives - pool creation, scheduled scrubs, and automatic snapshots
- **First VM** - spinning up an Ubuntu Server to start running services
- **LXC containers** - lightweight services without the overhead of a full VM
- **Jellyfin** - media server running on top of Proxmox

The environment is ready. Now it's time to configure and use it.
