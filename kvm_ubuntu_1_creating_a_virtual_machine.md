# KVM on Ubuntu — Creating a Virtual Machine
### Engineer's Extensive Reference | Topic: VM Creation Only
> Ubuntu 20.04 / 22.04 / 24.04 LTS · libvirt · virt-install · virt-manager  
> Scope: Pre-creation planning → virt-install → all install methods → network config at creation time

---

## Table of Contents

1. [Before You Create Anything — Preflight Checks](#1-before-you-create-anything--preflight-checks)
2. [Guest VM Deployment Considerations](#2-guest-vm-deployment-considerations)
3. [virt-install Fundamentals](#3-virt-install-fundamentals)
4. [Method 1 — Install from ISO Image](#4-method-1--install-from-iso-image)
5. [Method 2 — Import an Existing Disk Image](#5-method-2--import-an-existing-disk-image)
6. [Method 3 — Install from Network Location](#6-method-3--install-from-network-location)
7. [Method 4 — PXE Boot Install](#7-method-4--pxe-boot-install)
8. [Method 5 — Autoinstall / Preseed (Unattended)](#8-method-5--autoinstall--preseed-unattended)
9. [Method 6 — Cloud Image + cloud-init](#9-method-6--cloud-image--cloud-init)
10. [Network Configuration at Creation Time](#10-network-configuration-at-creation-time)
11. [Disk & Storage Options at Creation Time](#11-disk--storage-options-at-creation-time)
12. [virt-manager GUI — Creating VMs](#12-virt-manager-gui--creating-vms)
13. [Post-Creation Verification Checklist](#13-post-creation-verification-checklist)
14. [Common Creation Errors & Fixes](#14-common-creation-errors--fixes)
15. [Full Option Reference Table](#15-full-option-reference-table)
16. [Quick-Reference Cheat Sheet](#16-quick-reference-cheat-sheet)

---

## 1. Before You Create Anything — Preflight Checks

Never skip these. Failed VM creation almost always traces back to a skipped preflight step.

### 1.1 Confirm KVM Hardware Support

```bash
# Ubuntu's dedicated tool — clearest output
sudo apt install -y cpu-checker
kvm-ok

# ✅ Good output:
# INFO: /dev/kvm exists
# KVM acceleration can be used

# ❌ Bad output:
# INFO: /dev/kvm does not exist
# → Enable Intel VT-x or AMD-V in BIOS/UEFI, then reboot

# Manual check: vmx = Intel VT-x, svm = AMD-V
grep -Eoc '(vmx|svm)' /proc/cpuinfo
# Returns number of CPU threads with the flag; 0 = unsupported or BIOS-disabled
```

### 1.2 Confirm Required Packages Are Installed

```bash
# Install the full stack in one shot
sudo apt update && sudo apt install -y \
  qemu-kvm \
  libvirt-daemon-system \
  libvirt-clients \
  bridge-utils \
  virtinst \
  virt-manager \
  libguestfs-tools \
  libosinfo-bin \
  cloud-image-utils \
  ovmf                    # For UEFI guest support

# Verify virt-install is available
virt-install --version

# Verify virsh connects to the daemon
virsh version
```

### 1.3 Confirm Your User Has the Right Groups

```bash
# Without these groups, most virt-install and virsh commands need sudo
groups $USER
# Must include: libvirt   kvm

# Add if missing
sudo usermod -aG libvirt,kvm $USER
newgrp libvirt              # Apply in current session without logging out
```

### 1.4 Confirm libvirtd Is Running

```bash
# Ubuntu 20.04 — monolithic daemon
sudo systemctl status libvirtd

# Ubuntu 22.04/24.04 — socket-activated modular daemons
sudo systemctl status virtqemud.socket
sudo systemctl status virtnetworkd.socket

# Start and enable if inactive
sudo systemctl enable --now libvirtd          # 20.04
sudo systemctl enable --now virtqemud         # 22.04+
```

### 1.5 Confirm the Default Network Exists and Is Active

```bash
virsh net-list --all
# NAME      STATE    AUTOSTART   PERSISTENT
# default   active   yes         yes        ← This is what you need

# If missing or inactive:
sudo virsh net-start default
sudo virsh net-autostart default

# Inspect what the default network provides
virsh net-dumpxml default
# Shows: subnet (192.168.122.0/24), DHCP range, bridge name (virbr0)
```

### 1.6 Confirm Available Host Resources

```bash
# CPU
lscpu | grep -E 'CPU\(s\)|Model name|Thread|Core|Socket'

# Memory
free -h
# Headroom rule: always leave ≥ 2GB free for the host OS itself

# Disk space (default image path)
df -h /var/lib/libvirt/images
# Thin-provisioned qcow2 images grow, so ensure raw image capacity fits projected growth

# Confirm /dev/kvm permissions
ls -la /dev/kvm
# crw-rw---- 1 root kvm ... → Your user must be in the kvm group
```

### 1.7 Find the Correct --os-variant Value (Always Set This)

The `--os-variant` flag applies CPU pinning hints, timer settings, virtio driver defaults, and NIC models optimised for the target OS. Skipping it causes suboptimal performance.

```bash
# Search for Ubuntu variants
osinfo-query os | grep -i ubuntu
# ubuntu20.04   Ubuntu 20.04 LTS (Focal Fossa)
# ubuntu22.04   Ubuntu 22.04 LTS (Jammy Jellyfish)
# ubuntu24.04   Ubuntu 24.04 LTS (Noble Numbat)

# Other common variants
osinfo-query os | grep -i debian
osinfo-query os | grep -i centos
osinfo-query os | grep -i windows

# If osinfo-query isn't installed
sudo apt install libosinfo-bin
```

---

## 2. Guest VM Deployment Considerations

Think through these before running `virt-install`. Getting them wrong means recreating VMs later.

### 2.1 Performance Planning

Misaligned resource allocation is the #1 cause of bad VM performance.

```bash
# Understand NUMA topology before allocating vCPUs
numactl --hardware
lscpu --extended

# Example: 2-socket host with 8 cores per socket
# → Avoid giving a VM more vCPUs than one NUMA node has cores
# → A VM spanning NUMA nodes incurs memory latency penalties

# Check host memory pressure before deciding guest RAM
free -m
vmstat -s | head -5
```

**Rules of thumb:**

| Workload | vCPUs | RAM | Disk type |
|---|---|---|---|
| Dev / test | 1–2 | 1–2 GB | qcow2, cache=writeback |
| Web server | 2–4 | 2–4 GB | qcow2, cache=none |
| Database (MySQL/Postgres) | 4–8 | 8–32 GB | qcow2 or raw, cache=none, io=native |
| CI/build runners | 4–8 | 4–8 GB | qcow2, cache=none |
| Kubernetes node | 4+ | 8+ GB | qcow2, cache=none |

### 2.2 I/O Requirements

```bash
# Check what disk types are available on the host
lsblk -d -o NAME,ROTA,SIZE,MODEL
# ROTA=0 → SSD/NVMe   ROTA=1 → spinning HDD

# For high-IOPS guests (DBs, message queues), always use:
--disk ...,bus=virtio,cache=none,io=native
# virtio bus avoids emulation overhead
# cache=none bypasses host page cache (avoids double-buffering)
# io=native uses O_DIRECT for lowest latency
```

### 2.3 Storage Planning

```bash
# Check current usage on the images directory
du -sh /var/lib/libvirt/images/*
df -h /var/lib/libvirt/images

# Thin vs thick provisioning decision:
# qcow2 (default) → thin-provisioned; grows dynamically up to the specified size
#                   Monitor carefully — host disk fills silently
# raw             → thick-provisioned; allocates full size immediately
#                   Predictable but consumes space upfront

# Recommended: pre-check how much the image will actually grow
qemu-img info /var/lib/libvirt/images/myvm.qcow2
# Look at: "disk size" (actual used) vs "virtual size" (max allocated)
```

**Ubuntu-specific storage note:** Ubuntu cloud images ship as sparse qcow2 files (~600 MB on disk but 2–3 GB virtual). Always resize them before importing for production use.

### 2.4 Networking Requirements

Decide the network type before creating — changing it later requires stopping the VM and editing XML.

| Requirement | Use |
|---|---|
| Dev/test, outbound access only | NAT (`network=default`) |
| Full LAN visibility, inbound access | Bridged (`bridge=br0`) |
| Live migration between hosts | Bridged only — NAT will block migration |
| Isolated/air-gapped VM | `--network=none` |
| Multiple network segments | Multiple `--network` flags |

### 2.5 SCSI Pass-Through Requirements

If the guest needs raw SCSI LUN access (e.g., for clustered filesystems or iSCSI initiator testing):

```bash
# Two conditions must both be met:
# 1. The virtio disk must be backed by a whole block device (not a file/partition)
# 2. device='lun' must be set in the domain XML

# Check available block devices on the host
lsblk -o NAME,TYPE,SIZE,MOUNTPOINT | grep disk
```

```xml
<!-- Domain XML fragment for SCSI LUN pass-through on Ubuntu -->
<devices>
  <emulator>/usr/bin/qemu-system-x86_64</emulator>   <!-- Ubuntu path, not /usr/libexec/ -->
  <disk type='block' device='lun'>
    <driver name='qemu' type='raw' cache='none' io='native'/>
    <source dev='/dev/sdb'/>
    <target dev='sda' bus='virtio'/>
  </disk>
</devices>
```

---

## 3. virt-install Fundamentals

### 3.1 How virt-install Works

```
virt-install (CLI) → libvirt API → libvirtd daemon → QEMU/KVM process
                                         ↓
                                  /etc/libvirt/qemu/<vmname>.xml  (persisted config)
                                         ↓
                                  /var/lib/libvirt/images/<vmname>.qcow2  (disk)
```

### 3.2 Required Options (Must Always Specify)

| Option | What it does | Ubuntu example |
|---|---|---|
| `--name` | Unique name for the VM | `--name prod-db01` |
| `--memory` | RAM in MiB | `--memory 4096` |
| `--disk` or `--filesystem` | Storage for the guest | `--disk size=20` |
| Install source | One of: `--cdrom`, `--location`, `--pxe`, `--import`, `--boot` | `--cdrom /path/to/ubuntu.iso` |

> `--vcpus` and `--os-variant` are technically optional but should always be set.

### 3.3 Most Useful Optional Options

| Option | Effect |
|---|---|
| `--vcpus 4` | Assign 4 virtual CPUs |
| `--vcpus 4,maxvcpus=8` | 4 CPUs at boot, hotplug up to 8 |
| `--cpu host-passthrough` | Expose all host CPU features to guest (best perf; harder to migrate) |
| `--cpu host-model` | Safe passthrough — good balance for live migration |
| `--os-variant ubuntu22.04` | OS-specific optimisation hints |
| `--graphics vnc,listen=0.0.0.0` | Remote VNC access (assign port with `,port=5910`) |
| `--graphics spice` | SPICE console — better than VNC for desktop guests |
| `--graphics none` | Headless — serial console only |
| `--console pty,target_type=serial` | Attach a serial console device |
| `--noautoconsole` | Don't open the console automatically after launch |
| `--autostart` | Boot this VM whenever the host boots |
| `--noreboot` | Don't reboot after install completes (useful in scripts) |
| `--dry-run` | Validate the command and print the XML without creating anything |
| `--print-xml` | Print the domain XML that would be generated, then exit |
| `--cloud-init` | Inject cloud-init data at first boot (virt-install ≥ 3.0) |
| `--qemu-commandline` | Pass extra args directly to QEMU (escape hatch) |

### 3.4 How to Explore Options

```bash
# Full list of flags
virt-install --help

# All attributes for a specific option (append =?)
virt-install --disk=?
virt-install --network=?
virt-install --graphics=?
virt-install --cpu=?

# Man page — the most complete reference
man virt-install

# Validate a command before running it
virt-install [your full command] --dry-run --print-xml
```

### 3.5 Privileges

```bash
# System-level VMs (shared by all users, managed by libvirtd)
virt-install --connect qemu:///system ...     # default; requires libvirt group membership

# Session-level VMs (your user only, no root, limited features)
virt-install --connect qemu:///session ...    # no shared networking, limited hardware access

# Explicit sudo (always works; use when group membership isn't active yet)
sudo virt-install ...
```

---

## 4. Method 1 — Install from ISO Image

Use when you have a downloaded `.iso` and want a standard interactive install.

### 4.1 Minimal Command

```bash
virt-install \
  --name ubuntu-server01 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20 \
  --cdrom /var/lib/libvirt/boot/ubuntu-22.04-live-server-amd64.iso \
  --os-variant ubuntu22.04
```

### 4.2 What Each Flag Does (Annotated)

```bash
virt-install \
  --name ubuntu-server01 \          # VM name; also becomes the QEMU process label
  --memory 2048 \                   # 2048 MiB RAM
  --vcpus 2 \                       # 2 virtual CPUs
  --disk size=20 \                  # Create a new 20 GB qcow2 in default pool (/var/lib/libvirt/images/)
  --cdrom /path/to/ubuntu.iso \     # Mount this ISO as the virtual CD-ROM drive
  --os-variant ubuntu22.04          # Apply Ubuntu 22.04-specific optimisations
```

### 4.3 Production-Ready Command (More Complete)

```bash
virt-install \
  --name prod-web01 \
  --memory 4096 \
  --vcpus 4 \
  --cpu host-model \
  --disk path=/var/lib/libvirt/images/prod-web01.qcow2,\
size=50,format=qcow2,bus=virtio,cache=none,io=native \
  --cdrom /var/lib/libvirt/boot/ubuntu-22.04-live-server-amd64.iso \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --graphics vnc,listen=0.0.0.0,port=5910 \
  --noautoconsole \
  --autostart
```

### 4.4 Connect to the Installer Console

```bash
# If --noautoconsole was used, connect manually via:

# Option A: VNC (if --graphics vnc was set)
virt-viewer --connect qemu:///system prod-web01
# Or connect any VNC client to: HOST_IP:5910

# Option B: GUI via virt-manager
virt-manager &

# Option C: Serial console (only if installer has serial support)
virsh console prod-web01
# Press Enter if the screen is blank
# Exit: Ctrl+]
```

### 4.5 Download Ubuntu ISOs

```bash
# Ubuntu 22.04 LTS (Jammy) — live server installer
wget -P /var/lib/libvirt/boot \
  https://releases.ubuntu.com/22.04/ubuntu-22.04.4-live-server-amd64.iso

# Ubuntu 24.04 LTS (Noble)
wget -P /var/lib/libvirt/boot \
  https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso

# Verify checksum (critical for production)
sha256sum /var/lib/libvirt/boot/ubuntu-22.04.4-live-server-amd64.iso
# Compare against: https://releases.ubuntu.com/22.04/SHA256SUMS
```

### 4.6 Headless ISO Install (No GUI on Host)

```bash
virt-install \
  --name headless-vm01 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20 \
  --cdrom /var/lib/libvirt/boot/ubuntu-22.04-live-server-amd64.iso \
  --os-variant ubuntu22.04 \
  --graphics none \
  --console pty,target_type=serial \
  --extra-args "console=ttyS0,115200n8" \
  --noautoconsole

# Then attach the serial console
virsh console headless-vm01
```

> The Ubuntu live-server ISO supports serial console natively. The desktop ISO does not.

### 4.7 UEFI (SecureBoot) ISO Install

```bash
# Requires ovmf package: sudo apt install ovmf

virt-install \
  --name uefi-vm01 \
  --memory 4096 \
  --vcpus 2 \
  --disk size=30 \
  --cdrom /var/lib/libvirt/boot/ubuntu-22.04-live-server-amd64.iso \
  --os-variant ubuntu22.04 \
  --boot uefi \
  --network network=default,model=virtio
```

---

## 5. Method 2 — Import an Existing Disk Image

Use when you already have a ready-to-boot disk image: a previous VM's disk, a vendor image, or a cloned golden image. No OS installer runs — the image must be bootable as-is.

### 5.1 Minimal Command

```bash
virt-install \
  --name imported-vm01 \
  --memory 2048 \
  --vcpus 2 \
  --disk /path/to/existing/disk.qcow2 \
  --import \
  --os-variant ubuntu22.04
```

> `--import` tells virt-install: skip the installer, boot directly from the first disk. The image must already have a bootable OS on it.

### 5.2 Key Behaviour of --import

- The **first** `--disk` option is the boot device — order matters if specifying multiple disks
- The image is used in-place (not copied) — keep backups
- `--cdrom`, `--location`, `--pxe` cannot be used together with `--import`

### 5.3 Deploy from a Golden Image (Most Common Use Case)

```bash
# Step 1: Create a working copy of the golden image
# Never import the golden image directly — always copy first
sudo cp /var/lib/libvirt/images/golden-ubuntu22.qcow2 \
        /var/lib/libvirt/images/app-node01.qcow2

# Step 2: Optionally resize if the golden image is smaller than needed
sudo qemu-img resize /var/lib/libvirt/images/app-node01.qcow2 30G

# Step 3: Import and boot
virt-install \
  --name app-node01 \
  --memory 4096 \
  --vcpus 4 \
  --disk /var/lib/libvirt/images/app-node01.qcow2,bus=virtio \
  --import \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --noautoconsole
```

### 5.4 Import with a Copy-on-Write Backing File (Space-Efficient Clones)

Instead of copying the full golden image, create a COW overlay — only changed blocks are stored.

```bash
# Create a thin overlay backed by the golden image
sudo qemu-img create -f qcow2 \
  -b /var/lib/libvirt/images/golden-ubuntu22.qcow2 \
  -F qcow2 \
  /var/lib/libvirt/images/dev-node01.qcow2 \
  20G

# Import the overlay
virt-install \
  --name dev-node01 \
  --memory 2048 \
  --vcpus 2 \
  --disk /var/lib/libvirt/images/dev-node01.qcow2,bus=virtio \
  --import \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --noautoconsole
```

> ⚠️ The backing (golden) image must not be modified or moved after overlays are created — all overlays become corrupted.

### 5.5 Import with Multiple Disks

```bash
# First disk = boot device, second = data disk
virt-install \
  --name db-server01 \
  --memory 8192 \
  --vcpus 4 \
  --disk /var/lib/libvirt/images/db-server01-os.qcow2,bus=virtio \
  --disk /var/lib/libvirt/images/db-server01-data.qcow2,bus=virtio \
  --import \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --noautoconsole
```

### 5.6 Generalise an Imported Image Before Import (virt-sysprep)

If the source image was from another VM, clean machine-specific state before importing:

```bash
# While the image file is NOT attached to any running VM
sudo virt-sysprep -a /var/lib/libvirt/images/app-node01.qcow2 \
  --operations defaults,ssh-hostkeys,machine-id,net-hwaddr \
  --hostname app-node01

# Verify what sysprep will do (dry run)
sudo virt-sysprep -a /var/lib/libvirt/images/app-node01.qcow2 --list-operations
```

### 5.7 Bulk Import Provisioning Script

```bash
#!/bin/bash
# Provision N VMs from a single golden image using COW overlays

GOLDEN="/var/lib/libvirt/images/golden-ubuntu22.qcow2"
IMG_DIR="/var/lib/libvirt/images"
COUNT=5

for i in $(seq -f "%02g" 1 $COUNT); do
  NAME="worker-${i}"
  IMG="${IMG_DIR}/${NAME}.qcow2"

  echo "→ Creating ${NAME}..."

  # COW overlay — only changed blocks stored
  sudo qemu-img create -f qcow2 -b "$GOLDEN" -F qcow2 "$IMG" 20G

  sudo virt-install \
    --name "$NAME" \
    --memory 2048 \
    --vcpus 2 \
    --disk "${IMG},bus=virtio,cache=none" \
    --import \
    --os-variant ubuntu22.04 \
    --network network=default,model=virtio \
    --noautoconsole \
    --noreboot

  echo "✓ ${NAME} created"
done

echo "All VMs created. Starting..."
for i in $(seq -f "%02g" 1 $COUNT); do
  virsh start "worker-${i}"
done
```

---

## 6. Method 3 — Install from Network Location

Use when you have an Ubuntu/Debian mirror on your network and want to pull the installer tree over HTTP, FTP, or NFS.

### 6.1 How --location Works

`--location` points to the **root of an installation tree** — a directory containing `dists/`, `pool/`, and the kernel/initrd. It is not an ISO path.

> **Ubuntu live-server ISO caveat:** The modern Ubuntu live-server installer does not expose a classic installation tree. Use `--cdrom` with the ISO for Ubuntu 20.04+. `--location` works well with Debian netboot mirrors and older Ubuntu alternate/mini ISOs.

### 6.2 Debian Netboot (Works Natively with --location)

```bash
virt-install \
  --name debian-vm01 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20 \
  --location http://deb.debian.org/debian/dists/bookworm/main/installer-amd64/ \
  --os-variant debian12 \
  --extra-args "console=ttyS0,115200n8" \
  --graphics none \
  --console pty,target_type=serial
```

### 6.3 Ubuntu with a Local HTTP Mirror (Using Mini ISO)

```bash
# Download the mini (netboot) ISO — not the live-server ISO
wget http://archive.ubuntu.com/ubuntu/dists/jammy/main/installer-amd64/current/legacy-images/netboot/mini.iso
sudo mv mini.iso /var/lib/libvirt/boot/ubuntu-22.04-mini.iso

virt-install \
  --name ubuntu-netboot \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20 \
  --location /var/lib/libvirt/boot/ubuntu-22.04-mini.iso \
  --os-variant ubuntu22.04 \
  --extra-args "console=ttyS0,115200n8" \
  --graphics none \
  --console pty,target_type=serial
```

### 6.4 NFS-Hosted Installation Tree

```bash
# First mount (or use the path directly if already mounted)
sudo mount -t nfs 192.168.1.50:/exports/ubuntu22 /mnt/ubuntu-mirror

virt-install \
  --name ubuntu-nfs-install \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20 \
  --location /mnt/ubuntu-mirror \
  --os-variant ubuntu22.04 \
  --extra-args "console=ttyS0,115200n8" \
  --graphics none

# Or pass NFS path directly (requires libvirt NFS mount support)
virt-install \
  --name ubuntu-nfs-install \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20 \
  --location nfs:192.168.1.50:/exports/ubuntu22 \
  --os-variant ubuntu22.04
```

### 6.5 Pass Kernel Arguments via --extra-args

`--extra-args` passes parameters directly to the installer kernel — only works with `--location`.

```bash
# Serial console (essential for headless --location installs)
--extra-args "console=ttyS0,115200n8"

# Set a static IP for the installer itself (not the installed OS)
--extra-args "ip=192.168.1.100::192.168.1.1:255.255.255.0:myhost:ens3:none"

# Both combined
--extra-args "console=ttyS0,115200n8 ip=192.168.1.100::192.168.1.1:255.255.255.0:myhost:ens3:none"
```

---

## 7. Method 4 — PXE Boot Install

Use when your environment has a DHCP + TFTP + PXE infrastructure (common in data centres). The installer kernel and initrd are fetched from the network at boot time.

### 7.1 Requirements

| Component | Requirement |
|---|---|
| Guest networking | **Bridged** (`bridge=br0`) — PXE does not work over NAT |
| Host bridge | Must exist before running virt-install |
| PXE server | DHCP must respond with `next-server` and `filename` options |
| TFTP | Must serve the PXE bootloader, kernel, and initrd |

### 7.2 Create the Host Bridge First (Netplan)

```yaml
# /etc/netplan/01-bridge.yaml
network:
  version: 2
  ethernets:
    enp3s0:                        # Replace with your actual NIC: ip link show
      dhcp4: false
  bridges:
    br0:
      interfaces: [enp3s0]
      dhcp4: true                  # Bridge gets the host's IP via DHCP
      parameters:
        stp: false
        forward-delay: 0
```

```bash
sudo netplan apply
brctl show br0                     # Verify enp3s0 is enslaved to br0
ip addr show br0                   # Verify br0 has an IP
```

### 7.3 PXE Install Command

```bash
virt-install \
  --name ubuntu-pxe-vm01 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20 \
  --network bridge=br0,model=virtio \
  --pxe \
  --os-variant ubuntu22.04 \
  --noautoconsole
```

> `--network=bridge:br0` and `--network bridge=br0,model=virtio` are equivalent. The latter explicitly sets the virtio model — always prefer it.

### 7.4 PXE with UEFI (Required for Many Modern Distros)

```bash
# PXE + UEFI requires iPXE or GRUB2 network bootloader in your TFTP setup
# and ovmf installed on the host: sudo apt install ovmf

virt-install \
  --name ubuntu-pxe-uefi \
  --memory 4096 \
  --vcpus 2 \
  --disk size=30 \
  --network bridge=br0,model=virtio \
  --pxe \
  --boot uefi \
  --os-variant ubuntu22.04 \
  --noautoconsole
```

### 7.5 Watch PXE Boot Progress

```bash
# Connect to the graphical console to watch PXE negotiation
virt-viewer ubuntu-pxe-vm01

# Or watch VM state transitions
watch -n1 'virsh domstate ubuntu-pxe-vm01'

# Check that the VM got a DHCP lease on the bridge network
virsh net-dhcp-leases default        # Only works for NAT; for bridged, check your DHCP server logs
```

---

## 8. Method 5 — Autoinstall / Preseed (Unattended)

Ubuntu replaced Kickstart with **Autoinstall** starting with 20.04. It is cloud-init compatible and the standard for scripted Ubuntu installs.

### 8.1 How Autoinstall Works

```
virt-install + ISO + autoinstall kernel arg
        ↓
Ubuntu installer boots
        ↓
Reads user-data from: a local file, an HTTP server, or a cloud-init seed ISO
        ↓
Installs with zero interaction
        ↓
Reboots into a configured system
```

### 8.2 Serve Autoinstall Config via HTTP

```bash
# Create the config files
mkdir -p /tmp/autoinstall-serve

cat > /tmp/autoinstall-serve/user-data <<'EOF'
#cloud-config
autoinstall:
  version: 1
  locale: en_US.UTF-8
  keyboard:
    layout: us
  identity:
    hostname: ubuntu-auto01
    username: ubuntu
    password: "$6$rounds=4096$abc123$HASH"    # openssl passwd -6 yourpassword
  ssh:
    install-server: true
    authorized-keys:
      - "ssh-ed25519 AAAA... your-public-key"
    allow-pw: false
  storage:
    layout:
      name: lvm
  network:
    network:
      version: 2
      ethernets:
        ens3:
          dhcp4: true
  packages:
    - qemu-guest-agent
    - curl
    - vim
    - git
  late-commands:
    - systemctl enable qemu-guest-agent
    - sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /target/etc/ssh/sshd_config
EOF

touch /tmp/autoinstall-serve/meta-data        # Must exist, can be empty

# Start an HTTP server
cd /tmp/autoinstall-serve
python3 -m http.server 8080 &
HTTP_PID=$!
echo "HTTP server PID: $HTTP_PID"

# Get the host IP that the guest can reach (via NAT gateway)
HOST_IP=$(ip addr show virbr0 | awk '/inet /{print $2}' | cut -d/ -f1)
echo "Autoinstall URL: http://${HOST_IP}:8080/"
```

### 8.3 Autoinstall VM Creation Command

```bash
virt-install \
  --name ubuntu-auto01 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20,bus=virtio \
  --cdrom /var/lib/libvirt/boot/ubuntu-22.04-live-server-amd64.iso \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --extra-args "autoinstall ds=nocloud-net;s=http://${HOST_IP}:8080/" \
  --graphics none \
  --console pty,target_type=serial \
  --noautoconsole

# After launch, optionally follow the install log:
virsh console ubuntu-auto01
# The VM will reboot automatically when complete. Ctrl+] to detach.
```

### 8.4 Autoinstall via Seed ISO (No HTTP Server Needed)

```bash
# Build a seed ISO containing user-data and meta-data
sudo apt install cloud-image-utils

cloud-localds /tmp/seed.iso \
  /tmp/autoinstall-serve/user-data \
  /tmp/autoinstall-serve/meta-data

virt-install \
  --name ubuntu-auto02 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20,bus=virtio \
  --cdrom /var/lib/libvirt/boot/ubuntu-22.04-live-server-amd64.iso \
  --disk /tmp/seed.iso,device=cdrom \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --graphics none \
  --console pty,target_type=serial \
  --noautoconsole
```

> Two CDROMs are attached: the Ubuntu ISO and the seed ISO. The installer automatically finds the seed ISO via the `ds=nocloud` datasource.

### 8.5 Generate a Hashed Password

```bash
# Generate a SHA-512 hash for use in user-data
openssl passwd -6 'YourPlainTextPassword'
# Output: $6$rounds=5000$salt$hash...

# Or use Python
python3 -c "import crypt; print(crypt.crypt('YourPassword', crypt.mksalt(crypt.METHOD_SHA512)))"
```

### 8.6 Validate Your Autoinstall Config

```bash
# The Ubuntu installer validates autoinstall YAML at boot
# To pre-validate on the host:
sudo apt install cloud-init

cloud-init schema --config-file /tmp/autoinstall-serve/user-data
# "Valid cloud-config: /tmp/autoinstall-serve/user-data" → ready to use
```

### 8.7 Bulk Unattended Provisioning Script

```bash
#!/bin/bash
# Spin up N VMs with autoinstall, sequentially

ISO="/var/lib/libvirt/boot/ubuntu-22.04-live-server-amd64.iso"
USERDATA="/tmp/autoinstall-serve/user-data"
METAADATA="/tmp/autoinstall-serve/meta-data"
COUNT=3

# Start HTTP server
cd /tmp/autoinstall-serve
python3 -m http.server 8080 &
HTTP_PID=$!
HOST_IP=$(ip addr show virbr0 | awk '/inet /{print $2}' | cut -d/ -f1)

for i in $(seq -f "%02g" 1 $COUNT); do
  NAME="app-server-${i}"
  echo "→ Provisioning ${NAME}..."

  sudo virt-install \
    --name "$NAME" \
    --memory 4096 \
    --vcpus 2 \
    --disk size=30,bus=virtio,cache=none \
    --cdrom "$ISO" \
    --os-variant ubuntu22.04 \
    --network network=default,model=virtio \
    --extra-args "autoinstall ds=nocloud-net;s=http://${HOST_IP}:8080/" \
    --graphics none \
    --noautoconsole \
    --noreboot &

  sleep 5    # Stagger starts to avoid DHCP collisions
done

wait
kill $HTTP_PID
echo "All VMs provisioned."
```

---

## 9. Method 6 — Cloud Image + cloud-init

The fastest way to create Ubuntu VMs. Ubuntu cloud images are pre-installed, pre-configured, and boot in seconds. This is the recommended approach for any environment that needs repeatable, fast deployment.

### 9.1 How It Works

```
Download pre-built Ubuntu cloud image (.img / .qcow2)
        ↓
Resize the image to desired disk size
        ↓
Create a cloud-init seed ISO (user-data + meta-data)
        ↓
Boot with both the resized image and seed ISO attached
        ↓
cloud-init runs at first boot, applies user-data config
        ↓
Ready VM — SSH in within ~30 seconds of boot
```

### 9.2 Download Ubuntu Cloud Images

```bash
# Ubuntu 22.04 LTS (Jammy)
wget -P /var/lib/libvirt/images/base/ \
  https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Ubuntu 24.04 LTS (Noble)
wget -P /var/lib/libvirt/images/base/ \
  https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# Verify the download
sha256sum /var/lib/libvirt/images/base/jammy-server-cloudimg-amd64.img
# Compare against: https://cloud-images.ubuntu.com/jammy/current/SHA256SUMS

# Inspect the image
qemu-img info /var/lib/libvirt/images/base/jammy-server-cloudimg-amd64.img
# virtual size: ~2.2 GB (actual used: ~600 MB)
```

### 9.3 Prepare the Disk Image

Always work from a copy — never import the base image directly.

```bash
# Method A: Full copy (independent — base can be deleted)
sudo qemu-img convert -f qcow2 -O qcow2 \
  /var/lib/libvirt/images/base/jammy-server-cloudimg-amd64.img \
  /var/lib/libvirt/images/myvm.qcow2

sudo qemu-img resize /var/lib/libvirt/images/myvm.qcow2 20G

# Method B: COW overlay (thin, base must remain unchanged)
sudo qemu-img create -f qcow2 \
  -b /var/lib/libvirt/images/base/jammy-server-cloudimg-amd64.img \
  -F qcow2 \
  /var/lib/libvirt/images/myvm.qcow2 \
  20G
```

### 9.4 Create cloud-init Configuration

```bash
# --- user-data ---
cat > /tmp/user-data <<'EOF'
#cloud-config

hostname: myvm
manage_etc_hosts: true
fqdn: myvm.example.com

users:
  - name: ubuntu
    groups: [sudo, adm]
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - "ssh-ed25519 AAAA... your-public-key-here"

# Disable password auth entirely
ssh_pwauth: false

# Set timezone
timezone: Australia/Sydney

# Grow the root partition to fill the resized disk
growpart:
  mode: auto
  devices: [/]

resize_rootfs: true

# Packages to install at first boot
packages:
  - qemu-guest-agent
  - curl
  - git
  - vim
  - htop

package_upgrade: true

# Commands to run after packages are installed
runcmd:
  - systemctl enable --now qemu-guest-agent
  - echo "cloud-init done" >> /var/log/cloud-init-done.log

# Write extra files into the VM
write_files:
  - path: /etc/motd
    content: |
      Welcome to myvm — provisioned by cloud-init
    permissions: '0644'

# Reboot after all config is applied (optional)
power_state:
  mode: reboot
  timeout: 30
  condition: true
EOF

# --- meta-data ---
cat > /tmp/meta-data <<EOF
instance-id: myvm-$(date +%s)
local-hostname: myvm
EOF

# Build the seed ISO
cloud-localds /tmp/seed.iso /tmp/user-data /tmp/meta-data

# Verify
file /tmp/seed.iso
# /tmp/seed.iso: ISO 9660 CD-ROM filesystem data 'cidata'
```

### 9.5 Create the VM

```bash
virt-install \
  --name myvm \
  --memory 2048 \
  --vcpus 2 \
  --disk /var/lib/libvirt/images/myvm.qcow2,bus=virtio \
  --disk /tmp/seed.iso,device=cdrom \
  --import \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --graphics none \
  --noautoconsole

# Watch cloud-init progress
virsh console myvm
# Or wait and SSH in
```

### 9.6 Using --cloud-init Flag (virt-install ≥ 3.0)

Ubuntu 22.04 ships virt-install 4.x — the `--cloud-init` flag handles seed ISO creation internally.

```bash
# Generate an SSH keypair for this VM
ssh-keygen -t ed25519 -f ~/.ssh/kvm_ed25519 -N ""

virt-install \
  --name myvm-ci \
  --memory 2048 \
  --vcpus 2 \
  --disk /var/lib/libvirt/images/myvm-ci.qcow2 \
  --import \
  --os-variant ubuntu22.04 \
  --cloud-init ssh-key=~/.ssh/kvm_ed25519.pub,user-data=/tmp/user-data \
  --network network=default,model=virtio \
  --graphics none \
  --noautoconsole

# Check virt-install version
virt-install --version
```

### 9.7 Get the Guest IP and SSH In

```bash
# Via DHCP lease table (NAT network)
virsh net-dhcp-leases default

# Via guest agent (most reliable — requires qemu-guest-agent in guest)
virsh domifaddr myvm --source agent

# Via ARP (fallback — works while VM is still booting)
arp -n | grep $(virsh domiflist myvm | awk 'NR>2 && $1!="" {print $5}')

# SSH in
ssh ubuntu@<GUEST_IP> -i ~/.ssh/kvm_ed25519

# Check cloud-init completed successfully
ssh ubuntu@<GUEST_IP> 'cloud-init status --wait'
# cloud-init v. 23.x finished ... datasource DataSourceNoCloud
```

### 9.8 Cloud Image Bulk Deployment Script

```bash
#!/bin/bash
# Deploy N VMs from a cloud image using COW overlays + cloud-init

BASE_IMG="/var/lib/libvirt/images/base/jammy-server-cloudimg-amd64.img"
IMG_DIR="/var/lib/libvirt/images"
SSH_KEY="$HOME/.ssh/kvm_ed25519.pub"
COUNT=3
DISK_SIZE=20G

for i in $(seq -f "%02g" 1 $COUNT); do
  NAME="node-${i}"
  IMG="${IMG_DIR}/${NAME}.qcow2"
  SEED="/tmp/seed-${NAME}.iso"

  echo "→ Setting up ${NAME}..."

  # Create COW disk from base image
  sudo qemu-img create -f qcow2 -b "$BASE_IMG" -F qcow2 "$IMG" "$DISK_SIZE"

  # Create per-VM cloud-init config
  cat > /tmp/user-data-${NAME} <<EOF
#cloud-config
hostname: ${NAME}
manage_etc_hosts: true
users:
  - name: ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - "$(cat $SSH_KEY)"
packages: [qemu-guest-agent]
runcmd: [systemctl enable --now qemu-guest-agent]
EOF

  cat > /tmp/meta-data-${NAME} <<EOF
instance-id: ${NAME}-$(date +%s)
local-hostname: ${NAME}
EOF

  cloud-localds "$SEED" "/tmp/user-data-${NAME}" "/tmp/meta-data-${NAME}"

  sudo virt-install \
    --name "$NAME" \
    --memory 2048 \
    --vcpus 2 \
    --disk "${IMG},bus=virtio" \
    --disk "${SEED},device=cdrom" \
    --import \
    --os-variant ubuntu22.04 \
    --network network=default,model=virtio \
    --graphics none \
    --noautoconsole \
    --autostart &

done

wait
echo "✓ All $COUNT VMs provisioned."
echo "Use: virsh net-dhcp-leases default  to find IPs."
```

---

## 10. Network Configuration at Creation Time

### 10.1 Default NAT Network

```bash
# Explicit (identical to omitting --network entirely)
--network network=default,model=virtio

# What you get:
# - Guest IP: 192.168.122.x (assigned by libvirt's built-in DHCP)
# - Gateway: 192.168.122.1 (virbr0 on the host)
# - Outbound internet access: yes (via iptables MASQUERADE)
# - Inbound access from LAN: no (unless port-forwarding is set up)
```

```bash
# Check/start the default network
virsh net-list --all
virsh net-start default
virsh net-autostart default
virsh net-info default          # Shows: bridge=virbr0, subnet, DHCP range
```

### 10.2 Bridged Network with DHCP (Full LAN Access)

**Required for live migration. Required for inbound connections from the LAN.**

**Step 1 — Create the bridge on the host (Netplan):**

```yaml
# /etc/netplan/01-kvm-bridge.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:                      # Your physical NIC — find with: ip link show
      dhcp4: false
      dhcp6: false
  bridges:
    br0:
      interfaces: [enp3s0]
      dhcp4: true
      parameters:
        stp: false               # Disable STP — faster convergence for VMs
        forward-delay: 0
      mtu: 1500
```

```bash
sudo netplan apply
sudo netplan --debug apply       # Verbose if errors occur
ip link show br0                 # Must show: state UP
brctl show                       # Must show enp3s0 under br0
ip addr show br0                 # Must have an IP
```

**Step 2 — Use the bridge in virt-install:**

```bash
virt-install \
  --name bridged-vm01 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20,bus=virtio \
  --cdrom /var/lib/libvirt/boot/ubuntu-22.04-live-server-amd64.iso \
  --os-variant ubuntu22.04 \
  --network bridge=br0,model=virtio
```

### 10.3 Bridged Network with Static IP

Passes kernel arguments to the installer to configure a static IP during installation.

```bash
virt-install \
  --name static-vm01 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20,bus=virtio \
  --location /var/lib/libvirt/boot/ubuntu-22.04-mini.iso \
  --os-variant ubuntu22.04 \
  --network bridge=br0,model=virtio \
  --extra-args "ip=192.168.1.50::192.168.1.1:255.255.255.0:static-vm01.example.com:ens3:none"
```

**Kernel ip= argument format (memorise this):**

```
ip=<IP>::<gateway>:<netmask>:<hostname>:<interface>:<autoconf>
     ↑  ↑↑
     |  ||__ gateway (second field)
     |  |___ blank peer address (always empty)
     |_______ static IP for the guest
```

| Position | Value | Notes |
|---|---|---|
| 1 | `192.168.1.50` | Guest static IP |
| 2 | *(blank)* | Peer address — always empty |
| 3 | `192.168.1.1` | Default gateway |
| 4 | `255.255.255.0` | Subnet mask |
| 5 | `static-vm01.example.com` | Hostname |
| 6 | `ens3` | Guest NIC name (Ubuntu VMs typically use `ens3`) |
| 7 | `none` | Disable autoconfiguration |

> Ubuntu VMs usually see their first NIC as `ens3`. Verify inside a running VM: `ip link show`. If using a bridged NIC on virtio, it may be `enp1s0`.

### 10.4 No Network Interface

```bash
# Isolated VM with zero network connectivity
--network=none

# Use case: security-sensitive workloads, offline processing,
#           vulnerability assessment environments
```

### 10.5 Multiple Network Interfaces

```bash
# Two NICs: one NAT (management), one bridged (data/LAN)
virt-install \
  --name dual-nic-vm \
  --memory 4096 \
  --vcpus 2 \
  --disk size=20,bus=virtio \
  --import \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --network bridge=br0,model=virtio
# Guest will see: ens3 (NAT) and ens7/ens8 (bridge)
```

### 10.6 Custom NAT Network (Non-Default Subnet)

```bash
# Define a custom isolated NAT network
cat > /tmp/custom-net.xml <<EOF
<network>
  <name>custom-nat</name>
  <forward mode='nat'/>
  <bridge name='virbr1' stp='on' delay='0'/>
  <ip address='10.10.10.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.10.10.100' end='10.10.10.200'/>
    </dhcp>
  </ip>
</network>
EOF

virsh net-define /tmp/custom-net.xml
virsh net-start custom-nat
virsh net-autostart custom-nat

# Use it when creating a VM
--network network=custom-nat,model=virtio
```

### 10.7 VLAN-Tagged Network

```bash
# Create a VLAN interface on the host (requires vlan package)
sudo apt install vlan
sudo modprobe 8021q

# Create VLAN 100 on enp3s0
sudo ip link add link enp3s0 name enp3s0.100 type vlan id 100
sudo ip link set enp3s0.100 up

# Or via Netplan (persistent):
# In /etc/netplan/01-kvm-bridge.yaml, add:
#   vlans:
#     vlan100:
#       id: 100
#       link: enp3s0
#   bridges:
#     br-vlan100:
#       interfaces: [vlan100]
#       ...

# Then use the VLAN bridge in virt-install
--network bridge=br-vlan100,model=virtio
```

### 10.8 UFW Firewall — Don't Break Guest Networking

Ubuntu's UFW can inadvertently block VM traffic. Fix:

```bash
# Allow all traffic on the libvirt bridges
sudo ufw allow in on virbr0
sudo ufw allow out on virbr0
sudo ufw allow in on br0
sudo ufw allow out on br0

# Enable IP forwarding (required for NAT and bridged guest routing)
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-kvm-forward.conf
sudo sysctl -p /etc/sysctl.d/99-kvm-forward.conf

# Set UFW forward policy to ACCEPT
sudo sed -i 's/^DEFAULT_FORWARD_POLICY=.*/DEFAULT_FORWARD_POLICY="ACCEPT"/' \
  /etc/default/ufw
sudo ufw reload
```

---

## 11. Disk & Storage Options at Creation Time

### 11.1 --disk Option Anatomy

```bash
# Full form
--disk path=/var/lib/libvirt/images/myvm.qcow2,\
       size=20,\
       format=qcow2,\
       bus=virtio,\
       cache=none,\
       io=native,\
       sparse=true

# Shorthand (just size — creates qcow2 in default pool, virtio bus)
--disk size=20

# No disk
--disk none
```

### 11.2 Bus Types

| Bus | Guest device name | Performance | Notes |
|---|---|---|---|
| `virtio` | `/dev/vda`, `/dev/vdb`... | Best | Always use this for Ubuntu guests |
| `scsi` (virtio-scsi) | `/dev/sda`, `/dev/sdb`... | Good | Required for SCSI pass-through |
| `ide` | `/dev/hda`, `/dev/hdb`... | Poor | Legacy; avoid |
| `sata` | `/dev/sda`... | OK | For Windows guests; not Ubuntu |

### 11.3 Cache Modes — Choose Based on Workload

| Cache mode | Write behaviour | Risk on crash | Best for |
|---|---|---|---|
| `none` | Writes go to disk, bypassing host page cache | None | Production VMs, DBs |
| `writeback` | Writes cached in host RAM, flushed later | Data loss | Dev/test |
| `writethrough` | Writes to disk and host cache simultaneously | None | Safe but slow |
| `directsync` | Writes directly to disk, bypasses cache | None | Like none, stricter |
| `unsafe` | All caching, ignores flush requests | Very high | Fastest, CI/test only |

### 11.4 Create a Disk Image Manually Before Import

```bash
# Thin-provisioned qcow2 (default, grows as written)
qemu-img create -f qcow2 /var/lib/libvirt/images/myvm.qcow2 20G

# Pre-allocated raw (best performance, occupies full 20G immediately)
qemu-img create -f raw /var/lib/libvirt/images/myvm.raw 20G

# COW overlay on top of a base image
qemu-img create -f qcow2 \
  -b /var/lib/libvirt/images/base/ubuntu22.img \
  -F qcow2 \
  /var/lib/libvirt/images/overlay.qcow2 \
  20G

# Check image details
qemu-img info /var/lib/libvirt/images/myvm.qcow2
```

### 11.5 Pre-create Storage Using virsh Pool

```bash
# Create a volume in the default storage pool (better for libvirt management)
virsh vol-create-as default myvm.qcow2 20G --format qcow2

# Reference it in virt-install
--disk vol=default/myvm.qcow2,bus=virtio
```

---

## 12. virt-manager GUI — Creating VMs

virt-manager is the GTK graphical frontend for libvirt. Useful when attached to a desktop session on the host or connected via X11 forwarding.

### 12.1 Launch virt-manager

```bash
# On the host (requires a desktop environment)
virt-manager &

# Remotely via SSH X11 forwarding
ssh -X user@kvm-host virt-manager

# Remotely via SSH tunnel (no X11 required on client)
# Client: forward a local port to the host's VNC display
ssh -L 5900:localhost:5900 user@kvm-host
# Then connect a VNC client to localhost:5900
```

### 12.2 New VM Wizard Steps

1. **File → New Virtual Machine**
2. Choose install source:
   - Local install media (ISO) → browse to ISO file
   - Network install (URL) → enter mirror URL
   - Import existing disk image → browse to `.qcow2`
   - Network boot (PXE)
3. Choose OS: type "Ubuntu 22.04" — virt-manager auto-detects `--os-variant`
4. Set RAM and CPU
5. Configure storage: size, format, location
6. Name VM, choose network, click **Finish**

### 12.3 When to Use virt-manager vs virt-install

| Scenario | Use |
|---|---|
| One-off dev/test VM | virt-manager (GUI, faster for interactive) |
| Scripted/bulk provisioning | virt-install (scriptable) |
| Headless server (no GUI) | virt-install + virsh |
| Remote host, desktop available | virt-manager via SSH X11 |
| Production automation | virt-install in shell scripts or Ansible |

---

## 13. Post-Creation Verification Checklist

Run these after every VM creation to confirm the VM is healthy.

```bash
VM="your-vm-name"

# 1. VM is in the expected state
virsh domstate $VM
# → running

# 2. VM definition is persisted correctly
virsh dominfo $VM
# Shows: name, UUID, state, memory, CPUs, autostart status

# 3. Correct disk is attached
virsh domblklist $VM --details
# Shows: type, device, target, source path

# 4. Correct network is attached
virsh domiflist $VM
# Shows: interface, type, source, model, MAC

# 5. VM has a network lease (NAT network)
virsh net-dhcp-leases default
# Find the MAC from domiflist; confirm an IP is assigned

# 6. Guest is reachable via SSH
GUEST_IP=$(virsh net-dhcp-leases default | grep $(virsh domiflist $VM | awk 'NR>2{print $5}') | awk '{print $5}' | cut -d/ -f1)
ssh ubuntu@$GUEST_IP 'uptime && hostname && ip addr'

# 7. Guest agent is responding (if installed)
virsh domifaddr $VM --source agent
virsh guestinfo $VM

# 8. Autostart is set correctly (if needed)
virsh dominfo $VM | grep Autostart

# 9. Snapshot baseline (optional but recommended)
virsh snapshot-create-as $VM "baseline-clean" "Post-install baseline" --atomic
```

---

## 14. Common Creation Errors & Fixes

### ERROR: "Could not open '/dev/kvm': Permission denied"

```bash
# Cause: User not in the kvm group
sudo usermod -aG kvm $USER
newgrp kvm
# Or run with sudo
```

### ERROR: "Unable to connect to libvirt qemu:///system"

```bash
# Check if libvirtd is running
sudo systemctl status libvirtd
sudo systemctl start libvirtd

# Ubuntu 22.04+ — check the socket
sudo systemctl status virtqemud.socket
sudo systemctl enable --now virtqemud.socket

# Check group membership
groups $USER | grep libvirt
```

### ERROR: "No such file or directory" for ISO or disk path

```bash
# Fix file permissions so libvirt can read it
sudo chmod 644 /path/to/file.iso
sudo chown libvirt-qemu:kvm /var/lib/libvirt/images/*.qcow2

# Confirm AppArmor isn't blocking the path
sudo aa-status | grep qemu
journalctl -xe | grep apparmor | tail -20
# Fix: add the path to /etc/apparmor.d/abstractions/libvirt-qemu
```

### ERROR: "Domain with duplicate name already exists"

```bash
# Check existing VMs
virsh list --all

# Remove the conflicting entry (keeps disk)
virsh undefine conflicting-vm-name

# Or rename your new VM
--name different-name
```

### ERROR: "Unable to find suitable emulator"

```bash
# Cause: qemu-kvm not installed or wrong os-variant
sudo apt install qemu-kvm

# List available emulators
ls /usr/bin/qemu-system-*

# Try without --os-variant first to isolate the issue
virt-install ... --os-variant detect=off
```

### ERROR: "Network not found: no network with matching name 'default'"

```bash
virsh net-list --all
# If default is absent:
sudo virsh net-define /usr/share/libvirt/networks/default.xml
sudo virsh net-start default
sudo virsh net-autostart default
```

### ERROR: "Requested operation is not valid: network 'br0' is not active"

```bash
# The bridge doesn't exist on the host
ip link show br0        # Should show the bridge

# If missing, apply Netplan config
sudo netplan apply
ip link show br0        # Retry

# Alternatively — create the bridge manually for testing
sudo ip link add br0 type bridge
sudo ip link set enp3s0 master br0
sudo ip link set br0 up
sudo ip link set enp3s0 up
```

### ERROR: PXE boot times out / no lease

```bash
# Confirm the VM's NIC is on the bridged network (not NAT)
virsh domiflist my-pxe-vm
# Source should be br0, not virbr0

# Check if the bridge is forwarding packets
brctl show br0
sudo tcpdump -i br0 port 67 or port 68    # Watch for DHCP traffic

# Ensure no firewall is blocking UDP 67/68 on br0
sudo ufw allow in on br0 to any port 67 proto udp
sudo ufw allow in on br0 to any port 68 proto udp
```

### WARNING: "KVM not available; using TCG (software emulation)"

```bash
# This means hardware virtualisation is not being used — VERY slow
kvm-ok                         # Run preflight check
lsmod | grep kvm               # KVM module must be loaded
ls -la /dev/kvm                # Must exist and be accessible

# Load KVM module manually
sudo modprobe kvm_intel         # or kvm_amd
```

---

## 15. Full Option Reference Table

Key `virt-install` options relevant to VM creation:

| Option | Values / Format | Notes |
|---|---|---|
| `--name` | string | Unique per host |
| `--memory` | MiB integer | e.g. `2048` for 2 GB |
| `--vcpus` | integer or `n,maxvcpus=m` | |
| `--cpu` | `host-passthrough`, `host-model`, `EPYC`, etc. | `host-model` is safest for migration |
| `--os-variant` | `ubuntu22.04`, `debian12`, etc. | Get list: `osinfo-query os` |
| `--disk` | `path=,size=,format=,bus=,cache=,io=` | Most important option |
| `--filesystem` | `source,target` | For virtfs/9p sharing |
| `--cdrom` | `/path/to/file.iso` | ISO attach — cannot be physical drive |
| `--location` | `http://`, `ftp://`, `nfs:`, `/local/path` | Installation tree root |
| `--pxe` | (flag) | Requires bridged network |
| `--import` | (flag) | Skip installer; boot from first disk |
| `--boot` | `uefi`, `bios`, `cdrom,hd`, `kernel=,initrd=` | Post-install boot config |
| `--network` | `network=name`, `bridge=br0`, `none` | Multiple allowed |
| `--graphics` | `vnc,listen=0.0.0.0,port=N`, `spice`, `none` | |
| `--console` | `pty,target_type=serial` | Serial console device |
| `--extra-args` | kernel parameters | Only with `--location` |
| `--initrd-inject` | `/path/to/file` | Inject file into initrd (legacy Kickstart) |
| `--cloud-init` | `ssh-key=,user-data=,meta-data=` | virt-install ≥ 3.0 |
| `--noautoconsole` | (flag) | Don't open console automatically |
| `--autostart` | (flag) | Start VM on host boot |
| `--noreboot` | (flag) | Don't reboot after install |
| `--dry-run` | (flag) | Validate without creating |
| `--print-xml` | (flag) | Show generated XML and exit |
| `--connect` | `qemu:///system`, `qemu:///session` | Connection URI |

---

## 16. Quick-Reference Cheat Sheet

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PREFLIGHT (run before every VM creation)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 kvm-ok                              Hardware virtualisation check
 groups $USER | grep libvirt         Group membership check
 virsh net-list --all                Default network active?
 df -h /var/lib/libvirt/images       Enough disk space?
 osinfo-query os | grep ubuntu       Get --os-variant value

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 INSTALLATION METHODS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 ISO          --cdrom /path/to/ubuntu.iso
 Import       --import  (first --disk = boot device)
 Network      --location http://mirror/  (+ --extra-args for console)
 PXE          --network bridge=br0,model=virtio --pxe
 Autoinstall  --cdrom ubuntu.iso --extra-args "autoinstall ds=nocloud-net;s=http://HOST/"
 Cloud image  --import --disk overlay.qcow2 --disk seed.iso,device=cdrom
 cloud-init   --cloud-init ssh-key=~/.ssh/pub,user-data=/tmp/user-data  (virt-install ≥3.0)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 NETWORK OPTIONS AT CREATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 NAT (default)   --network network=default,model=virtio
 Bridged DHCP    --network bridge=br0,model=virtio
 Bridged static  --network bridge=br0,model=virtio \
                 --extra-args "ip=IP::GW:MASK:HOST:ens3:none"
 No network      --network=none
 Multi-NIC       --network network=default --network bridge=br0

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 DISK OPTIONS AT CREATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Simple          --disk size=20
 Full control    --disk path=/var/lib/libvirt/images/vm.qcow2,size=20,format=qcow2,bus=virtio,cache=none,io=native
 Raw block       --disk /dev/sdb,bus=virtio,cache=none,io=native
 No disk         --disk none

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 UBUNTU PACKAGES FOR VM CREATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Core            qemu-kvm libvirt-daemon-system libvirt-clients
 CLI tools       virtinst (contains virt-install, virt-clone, virt-xml)
 GUI             virt-manager virt-viewer
 OS info         libosinfo-bin  (osinfo-query)
 Cloud images    cloud-image-utils  (cloud-localds)
 UEFI            ovmf
 Guest tools     libguestfs-tools  (virt-sysprep, virt-customize)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 POST-CREATION CHECKS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 virsh domstate <vm>                      Is it running?
 virsh domblklist <vm> --details          Correct disk attached?
 virsh domiflist <vm>                     Correct NIC attached?
 virsh net-dhcp-leases default            Did it get an IP?
 virsh domifaddr <vm> --source agent      IP via guest agent
 ssh ubuntu@<IP> 'cloud-init status'      Did cloud-init finish?

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 KEY UBUNTU vs RHEL DIFFERENCES (VM creation context)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Packages        apt  (not yum)
 Package name    libvirt-daemon-system  (not libvirt)
 Package name    virtinst  (not virt-install package directly)
 QEMU binary     /usr/bin/qemu-system-x86_64  (not /usr/libexec/qemu-kvm)
 Network config  Netplan YAML  (not ifcfg-* scripts)
 Security        AppArmor  (not SELinux)
 Unattended      Autoinstall + cloud-init  (not Kickstart)
 NIC in guest    ens3 / enp1s0  (not always eth0)
 Grub update     update-grub  (not grub2-mkconfig)
 User groups     libvirt AND kvm  (not just libvirt)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

> **References:**
> - Ubuntu Server Guide — Virtualisation: https://ubuntu.com/server/docs/virtualization-libvirt
> - Ubuntu Autoinstall Reference: https://ubuntu.com/server/docs/install/autoinstall-reference
> - Ubuntu Cloud Images: https://cloud-images.ubuntu.com/
> - cloud-init documentation: https://cloudinit.readthedocs.io/
> - libvirt.org: https://libvirt.org/docs.html
> - `man virt-install` · `man cloud-localds` · `man qemu-img` · `man virsh`
