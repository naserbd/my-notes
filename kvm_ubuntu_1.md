# KVM Virtualization — Engineer's Reference Notes
> Ubuntu 20.04 / 22.04 LTS · libvirt · virt-manager · virt-install  
> Last updated: 2024 | For internal/secondary memory use
> Md Abdullah Al Naser

---

## Table of Contents
1. [KVM Architecture Overview](#1-kvm-architecture-overview)
2. [Host Setup & Prerequisites](#2-host-setup--prerequisites)
3. [Guest VM Deployment Considerations](#3-guest-vm-deployment-considerations)
4. [Creating VMs with virt-install](#4-creating-vms-with-virt-install)
5. [Installation Methods (with Examples)](#5-installation-methods-with-examples)
6. [Network Configuration During Guest Creation](#6-network-configuration-during-guest-creation)
7. [Storage Management](#7-storage-management)
8. [Managing VMs with virsh](#8-managing-vms-with-virsh)
9. [VM Lifecycle & Common Operations](#9-vm-lifecycle--common-operations)
10. [Performance Tuning Tips](#10-performance-tuning-tips)
11. [Snapshots & Cloning](#11-snapshots--cloning)
12. [Cloud Images & cloud-init](#12-cloud-images--cloud-init)
13. [AppArmor & Security](#13-apparmor--security)
14. [Troubleshooting Quick Reference](#14-troubleshooting-quick-reference)
15. [Quick-Reference Cheat Sheet](#15-quick-reference-cheat-sheet)

---

## 1. KVM Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│                   Host: Ubuntu LTS                   │
│                                                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐               │
│  │  Guest1 │  │  Guest2 │  │  Guest3 │  ← VMs        │
│  └────┬────┘  └────┬────┘  └────┬────┘               │
│       │            │            │                    │
│  ┌────▼────────────▼────────────▼──────┐             │
│  │        libvirt / QEMU-KVM           │ ← Hypervisor│
│  └─────────────────────────────────────┘             │
│  ┌─────────────────────────────────────┐             │
│  │         Linux Kernel + KVM Module   │             │
│  └─────────────────────────────────────┘             │
│  ┌─────────────────────────────────────┐             │
│  │           Physical Hardware         │             │
│  └─────────────────────────────────────┘             │
└──────────────────────────────────────────────────────┘
```

**Key Components:**

| Component | Role |
|---|---|
| `KVM` | Linux kernel module; turns the kernel into a type-1 hypervisor |
| `QEMU` | Emulates hardware devices; pairs with KVM for near-native speed |
| `libvirt` | Management API/daemon for KVM, QEMU, LXC, and more |
| `virsh` | CLI client for libvirt |
| `virt-manager` | GTK GUI frontend for libvirt |
| `virt-install` | CLI tool to provision and install VMs |
| `virt-viewer` | Displays graphical/VNC/SPICE console of a guest |
| `cloud-init` | Guest bootstrap tool; used heavily with Ubuntu cloud images |
| `AppArmor` | Ubuntu's MAC security layer (replaces SELinux on Ubuntu) |

> **Ubuntu vs RHEL difference:** Ubuntu uses `apt`, `netplan`, `ufw`, and `AppArmor` instead of `yum`, `ifcfg` scripts, `firewalld`, and `SELinux`. The QEMU binary is at `/usr/bin/qemu-system-x86_64`, not `/usr/libexec/qemu-kvm`. Ubuntu 20.04+ uses socket-activated libvirt (no need to manually start the daemon in most cases).

---

## 2. Host Setup & Prerequisites

### Verify CPU Virtualization Support

```bash
# Quick check — Ubuntu provides the kvm-ok tool (best method)
sudo apt install cpu-checker
kvm-ok
# Output if supported:
# INFO: /dev/kvm exists
# KVM acceleration can be used

# Manual check — look for vmx (Intel) or svm (AMD)
grep -Eoc '(vmx|svm)' /proc/cpuinfo
# Returns number of CPU threads with the flag; 0 = not supported / not enabled in BIOS
```

> If `kvm-ok` says KVM cannot be used, check your BIOS/UEFI settings and enable Intel VT-x or AMD-V.

### Install Virtualization Packages

```bash
# Recommended — install the full KVM/QEMU stack in one go
sudo apt update
sudo apt install -y \
  qemu-kvm \
  libvirt-daemon-system \
  libvirt-clients \
  bridge-utils \
  virtinst \
  virt-manager \
  libguestfs-tools \
  libosinfo-bin

# libvirt-daemon-system  → libvirtd service + default network
# libvirt-clients        → virsh and other CLI tools
# bridge-utils           → brctl for bridge management
# virtinst               → virt-install, virt-clone, virt-xml
# libguestfs-tools       → virt-sysprep, virt-customize, guestfish
# libosinfo-bin          → osinfo-query for OS variant names

# Optional: console viewer
sudo apt install -y virt-viewer

# Optional: performance/stats tooling
sudo apt install -y virt-top numactl
```

### Add Your User to the Required Groups

```bash
# libvirt  → allows managing VMs without sudo
# kvm      → allows direct access to /dev/kvm
sudo usermod -aG libvirt,kvm $USER

# Re-login or apply immediately in current session
newgrp libvirt
```

> On Ubuntu, `libvirt-daemon-system` creates the `libvirt` group automatically. Without group membership, you must prefix every virsh/virt-install call with `sudo`.

### Start & Enable libvirtd

```bash
# Ubuntu 20.04+ uses socket activation — daemon starts on demand.
# Explicitly enable/start if needed:
sudo systemctl enable --now libvirtd
sudo systemctl status libvirtd

# Ubuntu 22.04 uses modular libvirt daemons (virtqemud, etc.)
# The monolithic libvirtd still works but may show deprecation warnings.
sudo systemctl enable --now virtqemud   # Alternative on 22.04+
```

### Verify KVM Module is Loaded

```bash
lsmod | grep kvm
# Expected:
# kvm_intel   <size>  0     (Intel)
# kvm         <size>  1 kvm_intel
# — or —
# kvm_amd     <size>  0     (AMD)
# kvm         <size>  1 kvm_amd

# Load manually if missing
sudo modprobe kvm_intel    # or kvm_amd
```

### Check Host Capabilities

```bash
virsh capabilities          # Full XML report of host capabilities
virsh nodeinfo              # CPU count, memory, NUMA topology
virsh domcapabilities       # Domain-level features (machine types, etc.)
virsh version               # libvirt and QEMU versions
```

### Verify Default Network

```bash
virsh net-list --all
# Should show "default" as active. If inactive:
sudo virsh net-start default
sudo virsh net-autostart default
```

---

## 3. Guest VM Deployment Considerations

### Performance
- How many vCPUs does the workload need?
- How much RAM? Over-allocate slightly for DB/app servers.
- Does it need CPU pinning for latency-sensitive workloads?

```bash
# Check host CPU topology before planning vCPU allocation
lscpu
numactl --hardware    # Requires numactl package

# Check free host memory
free -h
```

### I/O Requirements
- High I/O (databases, log aggregation) → always use **virtio** drivers
- Large sequential I/O (backups, bulk writes) vs. small random I/O (OLTP) → affects cache strategy
- High I/O → `cache=none,io=native` on raw block devices

### Storage
- Monitor regularly to avoid thin-provisioning surprises:

```bash
du -sh /var/lib/libvirt/images/*     # Image file sizes
df -h /var/lib/libvirt/images        # Available host disk space
virsh domblkinfo <vmname> vda        # Guest disk capacity vs. allocation
```

- `qcow2` is the default on Ubuntu; it supports snapshots and thin provisioning
- Physical storage capacity is the hard ceiling for all virtual disks

### Networking
- Live migration **requires** bridged networking — NAT won't work
- VLANs can be tagged on bridge ports to isolate tenants
- Plan bandwidth needs per guest; Ubuntu guests default to the virtio NIC model

### SCSI Pass-Through Requirements

For SCSI LUN pass-through (virtio-scsi), two conditions must both be met:

1. The virtio drive must be backed by a **whole physical block device** (not a partition or image file)
2. The `device` attribute must be `lun` in the domain XML

```xml
<devices>
  <emulator>/usr/bin/qemu-system-x86_64</emulator>
  <disk type='block' device='lun'>
    <driver name='qemu' type='raw' cache='none' io='native'/>
    <source dev='/dev/sdb'/>
    <target dev='sda' bus='virtio'/>
  </disk>
</devices>
```

> **Ubuntu note:** The emulator path is `/usr/bin/qemu-system-x86_64`, not `/usr/libexec/qemu-kvm` as on RHEL.

---

## 4. Creating VMs with virt-install

### Key Required Options

| Option | Description | Example |
|---|---|---|
| `--name` | VM name (unique per host) | `--name webserver01` |
| `--memory` | RAM in MiB | `--memory 4096` |
| `--vcpus` | Virtual CPU count | `--vcpus 4` |
| `--disk` | Storage config | `--disk size=20` |
| `--os-variant` | OS hint for optimized config | `--os-variant ubuntu22.04` |

### Useful Optional Options

| Option | Description |
|---|---|
| `--cpu host-passthrough` | Expose full host CPU feature set to guest |
| `--graphics vnc,listen=0.0.0.0` | VNC console accessible from network |
| `--graphics spice` | SPICE console (better than VNC for desktop use) |
| `--graphics none` | Headless — serial console only |
| `--console pty,target_type=serial` | Attach a serial console |
| `--noautoconsole` | Don't auto-open console after launch |
| `--autostart` | Start VM on host boot |
| `--noreboot` | Don't reboot after install (good for scripting) |
| `--cloud-init` | Inject cloud-init user-data at first boot |

### Find the Correct OS Variant

The `--os-variant` flag feeds libvirt optimization hints. Always set it.

```bash
# Search for Ubuntu variants
osinfo-query os | grep -i ubuntu
# Output examples:
# ubuntu20.04   Ubuntu 20.04 LTS
# ubuntu22.04   Ubuntu 22.04 LTS
# ubuntu24.04   Ubuntu 24.04 LTS

# Search for other distros
osinfo-query os | grep -i debian
osinfo-query os | grep -i centos
```

### Get Help

```bash
virt-install --help
virt-install --disk=?         # Show all attributes for --disk
virt-install --network=?      # Show all attributes for --network
man virt-install
```

---

## 5. Installation Methods (with Examples)

### 5.1 From ISO Image

```bash
virt-install \
  --name ubuntu-server01 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20 \
  --cdrom /var/lib/libvirt/boot/ubuntu-22.04-live-server-amd64.iso \
  --os-variant ubuntu22.04
```

**Extended production example:**

```bash
virt-install \
  --name prod-web01 \
  --memory 4096 \
  --vcpus 2 \
  --cpu host-passthrough \
  --disk path=/var/lib/libvirt/images/prod-web01.qcow2,size=50,format=qcow2,bus=virtio,cache=none \
  --cdrom /var/lib/libvirt/boot/ubuntu-22.04-live-server-amd64.iso \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --graphics vnc,listen=0.0.0.0,port=5910 \
  --noautoconsole \
  --autostart
```

> `--cdrom` accepts a local path to an ISO. It cannot use a physical CD-ROM device.  
> Download Ubuntu ISOs: `wget https://releases.ubuntu.com/22.04/ubuntu-22.04-live-server-amd64.iso`

---

### 5.2 Import Existing Disk Image

Use when you have a pre-built, cloned, or downloaded cloud image.

```bash
virt-install \
  --name ubuntu-imported \
  --memory 2048 \
  --vcpus 2 \
  --disk /path/to/existing/disk.qcow2 \
  --import \
  --os-variant ubuntu22.04 \
  --noautoconsole
```

**Practical use case — deploy from a golden image:**

```bash
# Clone the golden image
sudo cp /var/lib/libvirt/images/golden-ubuntu22.qcow2 \
        /var/lib/libvirt/images/app-node01.qcow2

# Import and boot
virt-install \
  --name app-node01 \
  --memory 4096 \
  --vcpus 4 \
  --disk /var/lib/libvirt/images/app-node01.qcow2 \
  --import \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --noautoconsole
```

---

### 5.3 From Network Location (Debian/Ubuntu Mirror)

```bash
virt-install \
  --name ubuntu-netinstall \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20 \
  --location http://archive.ubuntu.com/ubuntu/dists/jammy/main/installer-amd64/ \
  --os-variant ubuntu22.04 \
  --extra-args "console=ttyS0,115200n8 --- console=ttyS0,115200n8"
```

> For Ubuntu live-server ISOs, `--location` is not supported directly; use `--cdrom` with the ISO instead.  
> `--location` works well with the traditional (non-live) Ubuntu server installer or Debian netboot trees.

---

### 5.4 PXE Boot

Requires a working DHCP + TFTP + PXE server on the bridged network.

```bash
virt-install \
  --name ubuntu-pxe \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20 \
  --network=bridge:br0,model=virtio \
  --pxe \
  --os-variant ubuntu22.04
```

> Both `--network=bridge:brX` and `--pxe` are required.  
> The bridge `br0` must already exist on the host. See Section 6.2 for bridge creation.

---

### 5.5 Preseed / Autoinstall (Ubuntu's Unattended Install)

Ubuntu replaced Kickstart with **Autoinstall** (20.04+) for the live-server installer.

**Autoinstall via cloud-init on a local HTTP server:**

```bash
# Serve the autoinstall config
python3 -m http.server 8080 &   # from dir containing user-data and meta-data

virt-install \
  --name ubuntu-auto \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20 \
  --cdrom /var/lib/libvirt/boot/ubuntu-22.04-live-server-amd64.iso \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --extra-args "autoinstall ds=nocloud-net;s=http://HOST_IP:8080/" \
  --noautoconsole
```

**Minimal `user-data` (autoinstall config):**

```yaml
#cloud-config
autoinstall:
  version: 1
  identity:
    hostname: ubuntu-server
    username: ubuntu
    password: "$6$xyz..."   # Use: openssl passwd -6 yourpassword
  storage:
    layout:
      name: lvm
  ssh:
    install-server: true
    authorized-keys:
      - "ssh-rsa AAAA... your-key"
  packages:
    - qemu-guest-agent
    - curl
  late-commands:
    - systemctl enable qemu-guest-agent
```

**`meta-data` file (can be empty but must exist):**

```yaml
instance-id: ubuntu-auto-01
local-hostname: ubuntu-server
```

**Bulk provisioning script:**

```bash
#!/bin/bash
BASE_IMG="/var/lib/libvirt/images/golden-ubuntu22.qcow2"
IMG_DIR="/var/lib/libvirt/images"

for i in 01 02 03; do
  NAME="app-node-${i}"
  IMG="${IMG_DIR}/${NAME}.qcow2"

  # Clone the golden image
  sudo qemu-img create -f qcow2 -b "$BASE_IMG" -F qcow2 "$IMG" 20G

  sudo virt-install \
    --name "$NAME" \
    --memory 2048 \
    --vcpus 2 \
    --disk "$IMG" \
    --import \
    --os-variant ubuntu22.04 \
    --network network=default,model=virtio \
    --noautoconsole \
    --noreboot
done
```

---

## 6. Network Configuration During Guest Creation

### 6.1 Default NAT Network

```bash
# Implicit — NAT is used when no --network flag is given
virt-install ...

# Explicit
virt-install ... --network network=default,model=virtio
```

**How NAT works:**

```
Guest (192.168.122.x) → virbr0 (192.168.122.1) → iptables MASQUERADE → Host → Internet
```

- Guests reach the outside world; outside cannot reach guests (without port-forwarding)
- Perfect for dev/test VMs
- The `default` network uses `192.168.122.0/24` by default

```bash
# Manage the default network
virsh net-list --all
virsh net-start default
virsh net-autostart default
virsh net-info default         # Show subnet, bridge name (virbr0), etc.
virsh net-dumpxml default      # Full network XML
```

---

### 6.2 Bridged Network with DHCP (Ubuntu — Netplan)

Bridged networking gives guests a real IP on your LAN — required for live migration.

**Create the bridge using Netplan (Ubuntu 18.04+):**

```yaml
# /etc/netplan/01-kvm-bridge.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:                   # Replace with your actual NIC name: ip link show
      dhcp4: false
  bridges:
    br0:
      interfaces: [enp3s0]
      dhcp4: true
      parameters:
        stp: false
        forward-delay: 0
```

```bash
sudo netplan apply
ip link show br0              # Verify bridge is up
brctl show                    # Verify enp3s0 is enslaved to br0
```

> **Find your NIC name:** `ip link show` or `ls /sys/class/net/`. Ubuntu uses predictable names like `enp3s0`, `ens3`, `eth0` (in VMs), or `eno1`.

```bash
# Use the bridge for a new VM
virt-install ... --network bridge=br0,model=virtio
```

---

### 6.3 Bridged Network with Static IP

```bash
virt-install ... \
  --network bridge=br0,model=virtio \
  --extra-args "ip=192.168.1.50::192.168.1.1:255.255.255.0:myhost.example.com:ens3:none"
```

**Format of the `ip=` kernel parameter:**

```
ip=<IP>::<gateway>:<netmask>:<hostname>:<interface>:<autoconf>
```

| Field | Example | Meaning |
|---|---|---|
| IP | `192.168.1.50` | Guest static IP |
| (empty) | `` | (peer address — leave blank) |
| Gateway | `192.168.1.1` | Default gateway |
| Netmask | `255.255.255.0` | Subnet mask |
| Hostname | `myhost.example.com` | Guest hostname |
| Interface | `ens3` | Guest NIC (Ubuntu typically uses `ens3` inside VMs) |
| Autoconf | `none` | Disable auto-config |

---

### 6.4 No Network Interface

```bash
virt-install ... --network=none
```

Use for isolated/air-gapped VMs (security testing, offline processing).

---

### 6.5 Multiple NICs

```bash
virt-install ... \
  --network network=default,model=virtio \
  --network bridge=br0,model=virtio
```

---

### 6.6 UFW Firewall Considerations

Ubuntu uses UFW, but libvirt manages its own iptables rules. Conflicts can occur.

```bash
# Allow KVM guest traffic through UFW
sudo ufw allow in on virbr0
sudo ufw allow out on virbr0

# Allow forwarding (required for NAT)
# In /etc/default/ufw: DEFAULT_FORWARD_POLICY="ACCEPT"
# In /etc/ufw/sysctl.conf: net/ipv4/ip_forward=1

sudo ufw reload
```

---

## 7. Storage Management

### Disk Image Formats

| Format | Use Case | Notes |
|---|---|---|
| `qcow2` | Default on Ubuntu | Snapshots, thin provisioning, compression |
| `raw` | Maximum performance | No overhead, no snapshots |
| `vmdk` | VMware interop | Usable but not native |

### Create Disk Images with qemu-img

```bash
# Create a 20GB qcow2 image (thin-provisioned)
qemu-img create -f qcow2 /var/lib/libvirt/images/myvm.qcow2 20G

# Create a raw image (pre-allocated, fastest)
qemu-img create -f raw /var/lib/libvirt/images/myvm.raw 20G

# Create a qcow2 backed by a base image (COW — copy on write)
qemu-img create -f qcow2 -b base.qcow2 -F qcow2 overlay.qcow2 20G

# Inspect an image
qemu-img info /var/lib/libvirt/images/myvm.qcow2

# Grow an image (online resize possible with guest tools)
qemu-img resize /var/lib/libvirt/images/myvm.qcow2 +10G

# Convert between formats
qemu-img convert -f qcow2 -O raw input.qcow2 output.raw
qemu-img convert -f raw -O qcow2 input.raw output.qcow2

# Check image for errors
qemu-img check /var/lib/libvirt/images/myvm.qcow2
```

### Disk Options in virt-install

```bash
# Thin-provisioned qcow2 on virtio bus (recommended default)
--disk path=/var/lib/libvirt/images/vm.qcow2,size=20,format=qcow2,bus=virtio

# Cache modes
--disk ...,cache=none           # Best for production; avoids double-caching
--disk ...,cache=writeback      # Faster but risks data loss on crash
--disk ...,cache=writethrough   # Safe, slower

# I/O mode (use with cache=none for max performance)
--disk ...,io=native            # O_DIRECT bypass — best perf on real block devices
--disk ...,io=threads           # Default QEMU threads — fine for image files

# No disk
--disk none
```

### Storage Pool Management

```bash
virsh pool-list --all                          # All pools
virsh pool-info default                        # Info on default pool
virsh pool-refresh default                     # Rescan pool for new images
virsh pool-start default                       # Start a stopped pool

# Create a new pool pointing to a directory
virsh pool-define-as mypool dir --target /data/vms
virsh pool-build mypool
virsh pool-start mypool
virsh pool-autostart mypool

# List volumes in a pool
virsh vol-list default

# Create a volume inside a pool
virsh vol-create-as default myvm.qcow2 20G --format qcow2

# Delete a volume
virsh vol-delete myvm.qcow2 --pool default
```

### Grow a Guest Disk Online

```bash
# Step 1: Resize the image file (host side)
sudo qemu-img resize /var/lib/libvirt/images/myvm.qcow2 +10G

# Step 2: Notify the guest (if running)
sudo virsh blockresize myvm /var/lib/libvirt/images/myvm.qcow2 30G

# Step 3: Inside the guest — grow the partition and filesystem
sudo growpart /dev/vda 1           # Grow partition (install cloud-guest-utils)
sudo resize2fs /dev/vda1           # ext4
# or
sudo xfs_growfs /                  # xfs
```

---

## 8. Managing VMs with virsh

### VM State Control

```bash
virsh list                     # Running VMs only
virsh list --all               # All VMs including stopped

virsh start <vmname>           # Start VM
virsh shutdown <vmname>        # Graceful shutdown (sends ACPI signal)
virsh destroy <vmname>         # Force off (like pulling the power)
virsh reboot <vmname>          # Reboot
virsh reset <vmname>           # Hard reset (no ACPI)
virsh suspend <vmname>         # Pause (freeze in RAM, CPU released)
virsh resume <vmname>          # Resume from pause
virsh autostart <vmname>       # Start on host boot
virsh autostart --disable <vmname>
```

### VM Information

```bash
virsh dominfo <vmname>         # State, memory, vCPUs, autostart status
virsh domiflist <vmname>       # Network interfaces and MACs
virsh domblklist <vmname>      # Block devices (disks, CDROMs)
virsh dommemstat <vmname>      # Memory stats (needs balloon driver in guest)
virsh cpu-stats <vmname>       # CPU time stats
virsh domstats <vmname>        # All stats in one shot
virsh vncdisplay <vmname>      # VNC port (if applicable)
```

### Console Access

```bash
# Serial console (must be enabled in guest — see Troubleshooting)
virsh console <vmname>
# Exit: press Ctrl+]

# Graphical console
virt-viewer <vmname>
virt-viewer --connect qemu:///system <vmname>

# SSH to guest (get IP first)
virsh domifaddr <vmname>       # Shows guest IPs (requires guest agent or DHCP lease lookup)
```

### Edit VM Configuration

```bash
virsh edit <vmname>                      # Opens XML in $EDITOR
virsh dumpxml <vmname>                   # Print XML to stdout
virsh dumpxml <vmname> > backup.xml      # Backup config
virsh define backup.xml                  # Restore/re-import config
virsh setmaxmem <vmname> 8192 --config   # Set max memory (in KiB, requires reboot)
```

### Attach / Detach Devices Live

```bash
# Attach a disk
virsh attach-disk <vmname> /path/to/extra.qcow2 vdb \
  --driver qemu --subdriver qcow2 --live --persistent

# Detach a disk
virsh detach-disk <vmname> vdb --live --persistent

# Attach a NIC
virsh attach-interface <vmname> network default \
  --model virtio --live --persistent

# Detach a NIC (use MAC address)
virsh detach-interface <vmname> network --mac 52:54:00:xx:xx:xx --live

# Change memory (balloon driver required in guest)
virsh setmem <vmname> 4194304 --live    # in KiB (4GB = 4*1024*1024)

# Change vCPU count (hotplug — guest must support it)
virsh setvcpus <vmname> 4 --live
```

---

## 9. VM Lifecycle & Common Operations

### Delete a VM

```bash
virsh destroy <vmname>                        # Force off if running
virsh undefine <vmname>                       # Remove from libvirt (keeps disk)
virsh undefine <vmname> --remove-all-storage  # Remove + delete disk images
```

### Live Migration (Requires Bridged Networking)

```bash
# Migrate to another KVM host (shared storage)
virsh migrate --live <vmname> qemu+ssh://dest-host.example.com/system

# Migrate with explicit destination URI (non-shared storage — slower)
virsh migrate --live --copy-storage-all <vmname> \
  qemu+ssh://dest-host.example.com/system

# Check migration status
virsh domjobinfo <vmname>
```

**Prerequisites:**

- Both hosts use **bridged networking** (not NAT)
- Shared or compatible storage (NFS, Ceph, SAN) — or use `--copy-storage-all`
- Same or compatible CPU models — use `--cpu host-model` for cross-version safety
- SSH key-based auth between hosts (no password)
- Firewall allows TCP ports **49152–49216** (migration data channels)
- Same libvirt version recommended

```bash
# On both hosts: allow migration ports
sudo ufw allow 49152:49216/tcp
```

---

## 10. Performance Tuning Tips

### Use Virtio Drivers (Always)

Paravirtualized virtio drivers are dramatically faster than emulated hardware.

```bash
# In virt-install
--disk ...,bus=virtio               # Virtio block (vda, vdb...)
--network ...,model=virtio          # Virtio NIC (will appear as ens3/eth0 inside guest)
```

In the guest, verify:

```bash
lspci | grep -i virtio
ls /sys/bus/virtio/drivers/
```

### Install QEMU Guest Agent (Important for Automation)

The guest agent enables host→guest communication (IP discovery, freeze/thaw for snapshots, etc.).

```bash
# Inside the Ubuntu guest
sudo apt install qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent

# On the host: check agent is connected
virsh domfsinfo <vmname>       # Requires agent
virsh guestinfo <vmname>       # OS info via agent
virsh domifaddr <vmname> --source agent   # IPs via agent
```

### CPU Pinning (Latency-Sensitive Workloads)

```bash
# View host CPU topology
virsh nodeinfo
lscpu --extended

# Pin vCPU 0 of VM to physical core 2
virsh vcpupin <vmname> 0 2 --live --config

# Pin vCPU 1 to physical core 3
virsh vcpupin <vmname> 1 3 --live --config

# Verify pinning
virsh vcpupin <vmname>
```

### Huge Pages

```bash
# On host: allocate 2MB huge pages (1024 pages = 2GB)
echo 1024 | sudo tee /proc/sys/vm/nr_hugepages

# Make permanent
echo "vm.nr_hugepages = 1024" | sudo tee -a /etc/sysctl.d/99-hugepages.conf
sudo sysctl -p /etc/sysctl.d/99-hugepages.conf

# Enable in VM XML (virsh edit <vmname>)
```
```xml
<memoryBacking>
  <hugepages/>
</memoryBacking>
```

### Disk I/O Best Practices

```bash
# For DB / high-I/O VMs (avoids double caching)
--disk path=...,format=qcow2,bus=virtio,cache=none,io=native

# For raw block devices (fastest possible)
--disk /dev/sdb,bus=virtio,cache=none,io=native
```

### Memory Ballooning

Allows libvirt to reclaim unused guest memory dynamically.

```xml
<!-- In domain XML (virsh edit) -->
<memballoon model='virtio'>
  <stats period='10'/>
</memballoon>
```

```bash
# Inside Ubuntu guest — balloon driver is loaded by default
lsmod | grep balloon
# virtio_balloon  ...
```

---

## 11. Snapshots & Cloning

### Snapshots (qcow2 only)

```bash
# Create an internal snapshot (captures RAM + disk if VM is running)
virsh snapshot-create-as <vmname> snap1 "Before apt upgrade" --atomic

# Create a disk-only snapshot (faster, no RAM state)
virsh snapshot-create-as <vmname> snap1 "Before apt upgrade" --disk-only --atomic

# List snapshots
virsh snapshot-list <vmname>

# Show snapshot details
virsh snapshot-info <vmname> snap1

# Revert to a snapshot (VM must be stopped for disk-only snapshots)
virsh snapshot-revert <vmname> snap1

# Delete a snapshot
virsh snapshot-delete <vmname> snap1
```

> **Tip:** Take snapshots before major changes (kernel upgrades, migrations, config changes). They are not a backup replacement — use `rsync` or block-level backups for that.

### Clone a VM

```bash
# VM must be shut down
virsh shutdown <vmname>
virsh domstate <vmname>     # Confirm: "shut off"

# Clone
virt-clone \
  --original <source-vmname> \
  --name <new-vmname> \
  --file /var/lib/libvirt/images/<new-vmname>.qcow2

# Auto-generate name and path
virt-clone --original <vmname> --auto-clone
```

> `virt-clone` regenerates MAC addresses automatically.

### Sysprep (Generalize a Clone Before Deployment)

```bash
# Clean up machine-specific state while VM is shut off
sudo virt-sysprep -d <vmname>

# Common operations performed:
# - Removes SSH host keys (regenerated on first boot)
# - Clears machine-id (/etc/machine-id)
# - Removes bash history, logs, temp files
# - Resets hostname

# On an offline image file
sudo virt-sysprep -a /var/lib/libvirt/images/myvm.qcow2

# Customize a clone (inject files, run commands)
sudo virt-customize -a /var/lib/libvirt/images/myvm.qcow2 \
  --hostname newhost \
  --run-command 'apt-get install -y nginx' \
  --ssh-inject ubuntu:file:/home/user/.ssh/id_rsa.pub
```

---

## 12. Cloud Images & cloud-init

Ubuntu cloud images are pre-built, minimal disk images — much faster to deploy than running an installer. This is the recommended approach for automated environments.

### Download Ubuntu Cloud Images

```bash
# Ubuntu 22.04 LTS cloud image (qcow2)
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Ubuntu 24.04 LTS
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# Move to libvirt images directory
sudo mv jammy-server-cloudimg-amd64.img /var/lib/libvirt/images/
```

### Deploy a Cloud Image VM with cloud-init

```bash
# Step 1: Create a working copy with desired size
sudo qemu-img create -f qcow2 \
  -b /var/lib/libvirt/images/jammy-server-cloudimg-amd64.img \
  -F qcow2 \
  /var/lib/libvirt/images/myvm.qcow2 \
  20G

# Step 2: Create cloud-init user-data
cat > /tmp/user-data <<EOF
#cloud-config
hostname: myvm
manage_etc_hosts: true
users:
  - name: ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - $(cat ~/.ssh/id_rsa.pub)
packages:
  - qemu-guest-agent
  - curl
  - git
runcmd:
  - systemctl enable --now qemu-guest-agent
EOF

# Step 3: Create empty meta-data
touch /tmp/meta-data

# Step 4: Build the seed ISO (cloud-init data disk)
sudo apt install cloud-image-utils
cloud-localds /tmp/seed.iso /tmp/user-data /tmp/meta-data

# Step 5: Import and boot with seed ISO
sudo virt-install \
  --name myvm \
  --memory 2048 \
  --vcpus 2 \
  --disk /var/lib/libvirt/images/myvm.qcow2,bus=virtio \
  --disk /tmp/seed.iso,device=cdrom \
  --import \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --noautoconsole
```

### Use virt-install's Built-in --cloud-init (virt-install 3.0+)

```bash
# Generate an SSH keypair if needed
ssh-keygen -t ed25519 -f ~/.ssh/kvm_id_ed25519

# One-liner cloud image deployment
sudo virt-install \
  --name myvm \
  --memory 2048 \
  --vcpus 2 \
  --disk /var/lib/libvirt/images/myvm.qcow2 \
  --import \
  --os-variant ubuntu22.04 \
  --cloud-init ssh-key=~/.ssh/kvm_id_ed25519.pub,user-data=/tmp/user-data \
  --network network=default,model=virtio \
  --noautoconsole
```

### Get the Guest IP Address

```bash
# Via guest agent (most reliable)
virsh domifaddr <vmname> --source agent

# Via DHCP lease table
virsh net-dhcp-leases default

# Via ARP (fallback)
arp -n | grep $(virsh domiflist <vmname> | awk '/vnet/{print $5}')
```

---

## 13. AppArmor & Security

Ubuntu uses **AppArmor** (not SELinux) to confine QEMU processes. Profiles live in `/etc/apparmor.d/`.

### Common AppArmor Situations

```bash
# Check if AppArmor is blocking a VM from starting
sudo aa-status | grep qemu
journalctl -xe | grep apparmor

# If a VM can't access a custom image path, add it to the profile
sudo nano /etc/apparmor.d/abstractions/libvirt-qemu
# Add a line like:
#   /data/vms/** rwk,

sudo apparmor_parser -r /etc/apparmor.d/abstractions/libvirt-qemu
sudo systemctl restart libvirtd
```

### Security Best Practices

```bash
# Use a dedicated user for libvirt management (not root)
sudo usermod -aG libvirt devops-user

# Isolate guest disk images from world-readable paths
sudo chmod 711 /var/lib/libvirt/images
sudo ls -la /var/lib/libvirt/images/   # Should be root:root 600 or similar

# Enable SecureBoot for guests (UEFI/OVMF)
sudo apt install ovmf
# In virt-install:
--boot uefi
# In domain XML:
# <loader readonly='yes' type='pflash'>/usr/share/OVMF/OVMF_CODE.fd</loader>

# Check guest security features
virsh dominfo <vmname> | grep -i security
```

---

## 14. Troubleshooting Quick Reference

### VM Won't Start

```bash
virsh start <vmname>                         # Read the error message
sudo journalctl -u libvirtd --since "5 min ago"
sudo cat /var/log/libvirt/qemu/<vmname>.log  # Detailed QEMU logs
sudo dmesg | tail -30                        # Kernel messages

# Common causes:
# 1. AppArmor blocking image path (see Section 13)
# 2. Image file permissions wrong
#    sudo chown libvirt-qemu:kvm /var/lib/libvirt/images/*.qcow2
# 3. Default network not started
#    virsh net-start default
```

### Can't Connect via Serial Console

The Ubuntu live-server ISO does not enable serial console by default. Enable it in the guest:

```bash
# Inside the guest
sudo systemctl enable serial-getty@ttyS0.service
sudo systemctl start serial-getty@ttyS0.service

# Or add to GRUB permanently
sudo nano /etc/default/grub
# GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8"
# GRUB_TERMINAL="console serial"
# GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"

sudo update-grub       # Ubuntu uses update-grub, NOT grub2-mkconfig
```

### Network Not Working in Guest

```bash
# Check libvirt's default network
virsh net-list --all
virsh net-start default
virsh net-autostart default

# Check iptables rules libvirt sets
sudo iptables -L FORWARD -n -v | grep virbr
sudo iptables -t nat -L -n -v | grep MASQUERADE

# Check IP forwarding is enabled
sysctl net.ipv4.ip_forward    # Should be 1

# Enable if not
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.d/99-kvm.conf
sudo sysctl -p /etc/sysctl.d/99-kvm.conf

# Restart libvirtd to re-apply iptables rules
sudo systemctl restart libvirtd
```

### Permission Denied Errors

```bash
# Add user to libvirt and kvm groups
sudo usermod -aG libvirt,kvm $USER
newgrp libvirt

# Fix image file ownership
sudo chown libvirt-qemu:kvm /var/lib/libvirt/images/*.qcow2
sudo chmod 660 /var/lib/libvirt/images/*.qcow2

# Fix AppArmor (see Section 13)
```

### Storage Pool Issues

```bash
virsh pool-list --all
virsh pool-start default
virsh pool-refresh default
virsh pool-info default

# Recreate the default pool if missing
virsh pool-define-as default dir --target /var/lib/libvirt/images
virsh pool-build default
virsh pool-start default
virsh pool-autostart default
```

### Check VM Resource Usage from Host

```bash
sudo apt install virt-top
virt-top                          # Top-like view for all VMs
virsh domstats <vmname>           # All metrics in one command
virsh dommemstat <vmname>         # Memory stats
virsh cpu-stats <vmname> --total  # CPU time
```

### libvirtd Socket vs Daemon (Ubuntu 22.04+)

```bash
# Ubuntu 22.04 split libvirtd into modular daemons
# If virsh gives "failed to connect" errors:
sudo systemctl status virtqemud.socket
sudo systemctl enable --now virtqemud.socket
sudo systemctl enable --now virtqemud

# Use the session URI for non-root access
virsh --connect qemu:///session list
```

---

## 15. Quick-Reference Cheat Sheet

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 UBUNTU-SPECIFIC PACKAGE NAMES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Core:       qemu-kvm libvirt-daemon-system libvirt-clients
 Tools:      virtinst libguestfs-tools libosinfo-bin
 GUI:        virt-manager virt-viewer
 Networking: bridge-utils
 Cloud:      cloud-image-utils
 Stats:      virt-top numactl cpu-checker

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 INSTALL METHODS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 ISO          --cdrom /path/to/ubuntu.iso
 Import       --import --disk /path/to/disk.qcow2
 Network      --location http://archive.ubuntu.com/...
 PXE          --network=bridge:br0 --pxe
 Autoinstall  --extra-args "autoinstall ds=nocloud-net;s=http://HOST/"
 Cloud image  --import + cloud-localds seed.iso / --cloud-init

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 NETWORK TYPES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 NAT          --network network=default,model=virtio
 Bridge+DHCP  --network bridge=br0,model=virtio
 Static       --network bridge=br0 --extra-args "ip=IP::GW:MASK:HOST:ens3:none"
 None         --network=none

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 VIRSH ESSENTIALS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 virsh list --all                 List all VMs
 virsh start <vm>                 Start VM
 virsh shutdown <vm>              Graceful stop
 virsh destroy <vm>               Force stop
 virsh console <vm>               Serial console (Ctrl+] to exit)
 virsh domifaddr <vm> --source agent   Get guest IP
 virsh edit <vm>                  Edit XML
 virsh dumpxml <vm>               Print XML
 virsh snapshot-create-as <vm> <name>  Snapshot
 virsh snapshot-revert <vm> <name>     Revert

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 QEMU-IMG
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 qemu-img create -f qcow2 disk.qcow2 20G
 qemu-img create -f qcow2 -b base.qcow2 -F qcow2 overlay.qcow2 20G
 qemu-img info disk.qcow2
 qemu-img resize disk.qcow2 +10G
 qemu-img convert -f raw -O qcow2 in.raw out.qcow2
 qemu-img check disk.qcow2

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 KEY UBUNTU DIFFERENCES FROM RHEL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Package mgr     apt  (not yum/dnf)
 Network config  Netplan  (not ifcfg scripts)
 Grub update     update-grub  (not grub2-mkconfig)
 Firewall        ufw  (not firewalld)
 MAC security    AppArmor  (not SELinux)
 QEMU binary     /usr/bin/qemu-system-x86_64
 NIC name in VM  ens3  (not eth0 in most cases)
 Unattended      Autoinstall/cloud-init  (not Kickstart)
 User groups     libvirt, kvm  (not just libvirt)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 DEFAULT FILE LOCATIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 VM images:      /var/lib/libvirt/images/
 VM XML configs: /etc/libvirt/qemu/
 QEMU logs:      /var/log/libvirt/qemu/<vmname>.log
 AppArmor:       /etc/apparmor.d/abstractions/libvirt-qemu
 Netplan config: /etc/netplan/
 Cloud images:   https://cloud-images.ubuntu.com/
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

> **References:**
> - Ubuntu Server Guide — KVM/libvirt: https://ubuntu.com/server/docs/virtualization-libvirt
> - Ubuntu Cloud Images: https://cloud-images.ubuntu.com/
> - Ubuntu Autoinstall Reference: https://ubuntu.com/server/docs/install/autoinstall-reference
> - libvirt.org documentation: https://libvirt.org/docs.html
> - `man virt-install` · `man virsh` · `man qemu-img` · `man cloud-localds`
