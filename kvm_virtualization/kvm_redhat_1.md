# KVM Virtualization — Engineer's Reference Notes
> RHEL 7 · libvirt · virt-manager · virt-install  
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
12. [Troubleshooting Quick Reference](#12-troubleshooting-quick-reference)
13. [Quick-Reference Cheat Sheet](#13-quick-reference-cheat-sheet)

---

## 1. KVM Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│                   Host: RHEL 7                       │
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
| `KVM` (Kernel-based VM) | Linux kernel module that turns the kernel into a hypervisor |
| `QEMU` | Emulates hardware devices; works with KVM for near-native performance |
| `libvirt` | Management layer / API for KVM, QEMU, and other hypervisors |
| `virsh` | CLI tool to interact with libvirt |
| `virt-manager` | GUI frontend for libvirt |
| `virt-install` | CLI tool to create and install VMs |
| `virt-viewer` | Displays graphical console of a VM |

> **KVM vs full emulation:** KVM runs guest code directly on the CPU (hardware-assisted virtualization via Intel VT-x / AMD-V), making it dramatically faster than pure emulation.

---

## 2. Host Setup & Prerequisites

### Verify CPU Virtualization Support

```bash
# Check for Intel VT-x or AMD-V support
grep -E 'vmx|svm' /proc/cpuinfo | head -5

# vmx = Intel VT-x
# svm = AMD-V
# Empty output means hardware virtualization is disabled in BIOS
```

### Install Virtualization Packages

```bash
# Core group install (recommended)
yum groupinstall "Virtualization Host"

# Or install individual packages
yum install qemu-kvm qemu-img libvirt libvirt-python libguestfs-tools virt-install

# For GUI management
yum install virt-manager virt-viewer

# For NAT networking (needed for default network)
yum install libvirt-daemon-config-network
```

### Start & Enable libvirtd

```bash
systemctl start libvirtd
systemctl enable libvirtd
systemctl status libvirtd
```

### Verify KVM Module is Loaded

```bash
lsmod | grep kvm
# Expected output:
# kvm_intel   <size>  0        (or kvm_amd for AMD)
# kvm         <size>  1 kvm_intel
```

### Check Host Capabilities

```bash
virsh capabilities          # Full XML capability report
virsh nodeinfo              # CPU, memory, NUMA topology
virsh domcapabilities       # Domain-level capabilities
```

---

## 3. Guest VM Deployment Considerations

Before deploying any VM, answer these questions:

### Performance
- How many vCPUs does the workload need?
- How much RAM? (Rule of thumb: over-allocate slightly for DB/app servers)
- Does it need CPU pinning for latency-sensitive workloads?

```bash
# Check available host CPUs and NUMA topology
lscpu
numactl --hardware
```

### I/O Requirements
- High I/O workloads (databases, logging) → prefer **virtio** drivers over emulated IDE
- Consider disk block size: large sequential I/O (backups) vs. small random I/O (OLTP DBs)
- High I/O → use `cache=none` with `io=native` for raw block devices

### Storage
- Monitor storage regularly:

```bash
du -sh /var/lib/libvirt/images/*   # Check image sizes
df -h                               # Host disk space
virsh domblkinfo <vmname> vda       # Disk capacity/allocation for a guest
```

- Physical storage limits your virtual storage options — thin provisioning hides this until it's too late
- Always read the RHEL Virtualization Security Guide for storage isolation considerations

### Networking
- Consider bandwidth and latency requirements per guest
- Live migration requires bridged networking (not NAT)
- VLANs can be used to isolate guest traffic

### SCSI Request Requirements (Important!)

SCSI pass-through (virtio-scsi) requires:
1. Virtio drives must be backed by **whole physical disks** (not partitions or files)
2. The `device` parameter must be set to `lun` in the domain XML

```xml
<devices>
  <emulator>/usr/libexec/qemu-kvm</emulator>
  <disk type='block' device='lun'>
    <driver name='qemu' type='raw' cache='none' io='native'/>
    <source dev='/dev/sdb'/>
    <target dev='sda' bus='virtio'/>
  </disk>
</devices>
```

---

## 4. Creating VMs with virt-install

### Key Required Options

| Option | Description | Example |
|---|---|---|
| `--name` | VM name (must be unique) | `--name webserver01` |
| `--memory` | RAM in MiB | `--memory 4096` |
| `--vcpus` | Virtual CPU count | `--vcpus 4` |
| `--disk` | Storage config | `--disk size=20` |
| `--os-variant` | OS optimization hints | `--os-variant rhel7` |

### Useful Optional Options

| Option | Description |
|---|---|
| `--cpu host` | Expose host CPU features to guest (better performance) |
| `--graphics vnc` | Use VNC for graphical console |
| `--graphics none` | No graphical console (headless) |
| `--console pty,target_type=serial` | Serial console access |
| `--noautoconsole` | Don't automatically connect to the console |
| `--autostart` | Start VM automatically when host boots |
| `--noreboot` | Don't reboot after install (useful for scripting) |

### Get Help

```bash
virt-install --help                  # All options
virt-install --option=?              # Attributes for a specific option
man virt-install                     # Full man page

# List valid OS variants
osinfo-query os | grep rhel          # Requires osinfo-db package
virt-install --os-variant list       # Older method
```

---

## 5. Installation Methods (with Examples)

### 5.1 From ISO Image

```bash
virt-install \
  --name guest1-rhel7 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=8 \
  --cdrom /path/to/rhel7.iso \
  --os-variant rhel7
```

> The `--cdrom` option accepts: local ISO path or a URL to a minimal boot ISO.  
> **Cannot** use a physical CD-ROM/DVD-ROM device here.

**Extended example with extra options:**

```bash
virt-install \
  --name prod-web01 \
  --memory 4096 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/prod-web01.qcow2,size=50,format=qcow2,bus=virtio \
  --cdrom /isos/rhel7.9.iso \
  --os-variant rhel7 \
  --network network=default \
  --graphics vnc,listen=0.0.0.0 \
  --noautoconsole \
  --autostart
```

---

### 5.2 Import Existing Disk Image

Use when you already have a pre-built or cloned disk image.

```bash
virt-install \
  --name guest1-rhel7 \
  --memory 2048 \
  --vcpus 2 \
  --disk /path/to/imported/disk.qcow2 \
  --import \
  --os-variant rhel7
```

> `--import` skips OS installation entirely. The first `--disk` listed becomes the boot device.

**Practical use case — spin up a golden image clone:**

```bash
# First, clone the golden image
cp /var/lib/libvirt/images/golden-rhel7.qcow2 \
   /var/lib/libvirt/images/clone-app01.qcow2

# Then import it
virt-install \
  --name app01 \
  --memory 2048 \
  --vcpus 2 \
  --disk /var/lib/libvirt/images/clone-app01.qcow2 \
  --import \
  --os-variant rhel7 \
  --noautoconsole
```

---

### 5.3 From Network Location

```bash
virt-install \
  --name guest1-rhel7 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=8 \
  --location http://example.com/path/to/os \
  --os-variant rhel7
```

> `--location` supports: HTTP, FTP, NFS paths. Points to the installation tree root (not an ISO).  
> Useful for environments with a central repo/mirror server.

**With NFS example:**

```bash
virt-install \
  --name guest1-rhel7 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20 \
  --location nfs:192.168.1.10:/exports/rhel7 \
  --os-variant rhel7
```

---

### 5.4 PXE Boot

Requires a bridged network and a working PXE/TFTP server on the network.

```bash
virt-install \
  --name guest1-rhel7 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=8 \
  --network=bridge:br0 \
  --pxe \
  --os-variant rhel7
```

> **Both** `--network=bridge:brX` AND `--pxe` are required together.  
> The bridge (`br0`) must exist on the host before running this command.

---

### 5.5 Kickstart (Unattended Install)

Fully automated, no user interaction required during install.

```bash
virt-install \
  --name guest1-rhel7 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=8 \
  --location http://example.com/path/to/os \
  --os-variant rhel7 \
  --initrd-inject /path/to/ks.cfg \
  --extra-args="ks=file:/ks.cfg console=tty0 console=ttyS0,115200n8"
```

> `--initrd-inject` pushes the kickstart file into the initrd.  
> `--extra-args` passes kernel parameters including `ks=file:/ks.cfg` to point to it.

**Minimal kickstart file example (`ks.cfg`):**

```kickstart
#version=RHEL7
install
text
url --url="http://example.com/path/to/os"

lang en_US.UTF-8
keyboard us
timezone UTC --isUtc

rootpw --plaintext changeme
authconfig --enableshadow --passalgo=sha512

firewall --disabled
selinux --permissive

bootloader --location=mbr
clearpart --all --initlabel
autopart

%packages
@core
%end

%post
# Any post-install scripts here
%end

reboot
```

**For fully scripted bulk VM provisioning:**

```bash
#!/bin/bash
for i in 01 02 03; do
  virt-install \
    --name "app-node-${i}" \
    --memory 2048 \
    --vcpus 2 \
    --disk size=20 \
    --location http://repo.example.com/rhel7 \
    --os-variant rhel7 \
    --initrd-inject /opt/ks/ks-node.cfg \
    --extra-args="ks=file:/ks-node.cfg console=ttyS0,115200n8" \
    --network network=default \
    --noautoconsole \
    --noreboot
done
```

---

## 6. Network Configuration During Guest Creation

### 6.1 Default NAT Network

```bash
# Implicit (NAT is default when no --network is given)
virt-install ... 

# Explicit
virt-install ... --network default
```

**How NAT works:**

```
Guest (10.0.2.x) → libvirt NAT bridge (virbr0) → Host IP → External Network
```

- Guests can reach the outside world
- Outside world cannot directly reach guests (unless port forwarding is set up)
- Easiest option for dev/test VMs
- Requires: `libvirt-daemon-config-network` package + `default` network active

```bash
# Check/start the default network
virsh net-list --all
virsh net-start default
virsh net-autostart default
```

---

### 6.2 Bridged Network with DHCP

```bash
virt-install ... --network br0
```

- Guest gets an IP directly from the LAN's DHCP server
- Guest is fully visible on the LAN (like a physical machine)
- **Required for live migration**
- The bridge `br0` must be created on the host before this works

**Create the bridge on the host (RHEL 7):**

```bash
# /etc/sysconfig/network-scripts/ifcfg-br0
TYPE=Bridge
BOOTPROTO=dhcp
DEVICE=br0
ONBOOT=yes
DELAY=0

# /etc/sysconfig/network-scripts/ifcfg-eth0
TYPE=Ethernet
BOOTPROTO=none
DEVICE=eth0
ONBOOT=yes
BRIDGE=br0
```

```bash
systemctl restart network
brctl show    # Verify bridge exists
```

---

### 6.3 Bridged Network with Static IP

```bash
virt-install ... \
  --network br0 \
  --extra-args "ip=192.168.1.2::192.168.1.1:255.255.255.0:hostname.example.com:eth0:none"
```

**Format of the `ip=` argument:**

```
ip=<IP>::<gateway>:<netmask>:<hostname>:<interface>:<autoconf>
```

| Field | Example | Meaning |
|---|---|---|
| IP | `192.168.1.2` | Guest static IP |
| Gateway | `192.168.1.1` | Default gateway |
| Netmask | `255.255.255.0` | Subnet mask |
| Hostname | `test.example.com` | Guest hostname |
| Interface | `eth0` | Guest NIC to configure |
| Autoconf | `none` | Disable auto-config |

---

### 6.4 No Network Interface

```bash
virt-install ... --network=none
```

Use for isolated VMs (security scanning, offline processing, etc.)

---

### 6.5 Multiple NICs

```bash
virt-install ... \
  --network network=default \
  --network bridge=br0
```

---

## 7. Storage Management

### Disk Image Formats

| Format | Use Case | Notes |
|---|---|---|
| `raw` | Best performance | No compression, no snapshots |
| `qcow2` | Most flexible | Supports snapshots, thin provisioning, compression |
| `vmdk` | VMware interop | Can be used but not native |

### Create Disk Images with qemu-img

```bash
# Create a 20GB qcow2 image
qemu-img create -f qcow2 /var/lib/libvirt/images/myvm.qcow2 20G

# Create a raw image
qemu-img create -f raw /var/lib/libvirt/images/myvm.raw 20G

# Check image info
qemu-img info /var/lib/libvirt/images/myvm.qcow2

# Resize an existing image (grow only; shrinking is risky)
qemu-img resize /var/lib/libvirt/images/myvm.qcow2 +10G

# Convert between formats
qemu-img convert -f raw -O qcow2 input.raw output.qcow2
```

### Disk Options in virt-install

```bash
# Thin-provisioned qcow2, virtio bus
--disk path=/var/lib/libvirt/images/vm.qcow2,size=20,format=qcow2,bus=virtio

# Cache options
--disk path=...,cache=none          # Best for production (avoids double caching)
--disk path=...,cache=writeback     # Better performance, risk of data loss on crash
--disk path=...,cache=writethrough  # Safe but slower

# I/O mode
--disk path=...,io=native           # Use O_DIRECT (best with cache=none)
--disk path=...,io=threads          # Default QEMU threads

# No disk
--disk none
```

---

## 8. Managing VMs with virsh

`virsh` is the primary CLI for managing VMs after creation.

### VM State Control

```bash
virsh list                    # Running VMs
virsh list --all              # All VMs (including stopped)

virsh start <vmname>          # Start a VM
virsh shutdown <vmname>       # Graceful shutdown (sends ACPI signal)
virsh destroy <vmname>        # Force off (like pulling power)
virsh reboot <vmname>         # Reboot
virsh suspend <vmname>        # Pause (freeze in RAM)
virsh resume <vmname>         # Resume from pause
virsh autostart <vmname>      # Start on host boot
virsh autostart --disable <vmname>
```

### VM Information

```bash
virsh dominfo <vmname>         # General info (state, memory, CPUs)
virsh domiflist <vmname>       # Network interfaces
virsh domblklist <vmname>      # Block devices (disks)
virsh dommemstat <vmname>      # Memory statistics
virsh cpu-stats <vmname>       # CPU statistics
virsh vncdisplay <vmname>      # VNC connection details
```

### Console Access

```bash
virsh console <vmname>                  # Serial console (needs ttyS0 configured in guest)
virt-viewer <vmname>                    # Graphical console
# Exit virsh console: Ctrl+]
```

### Edit VM XML Configuration

```bash
virsh edit <vmname>            # Opens in $EDITOR
virsh dumpxml <vmname>         # Print current XML
virsh dumpxml <vmname> > vm-backup.xml   # Export/backup
virsh define vm-backup.xml     # Re-import/restore
```

### Attach/Detach Devices Live

```bash
# Attach a disk
virsh attach-disk <vmname> /path/to/disk.qcow2 vdb --driver qemu --subdriver qcow2 --live

# Detach a disk
virsh detach-disk <vmname> vdb --live

# Attach a network interface
virsh attach-interface <vmname> bridge br0 --live

# Set memory (must have ballooning driver in guest)
virsh setmem <vmname> 4096 --live    # in KiB

# Set vCPUs (hotplug; guest must support it)
virsh setvcpus <vmname> 4 --live
```

---

## 9. VM Lifecycle & Common Operations

### Delete a VM

```bash
# Stop the VM first
virsh destroy <vmname>

# Undefine (removes from libvirt, keeps disk)
virsh undefine <vmname>

# Undefine AND delete storage
virsh undefine <vmname> --remove-all-storage

# Manually remove disk image
rm /var/lib/libvirt/images/<vmname>.qcow2
```

### Live Migration (Requires Bridged Networking)

```bash
# Migrate to another host (shared storage assumed)
virsh migrate --live <vmname> qemu+ssh://destination-host/system

# With explicit destination URI
virsh migrate --live --persistent <vmname> \
  qemu+ssh://dest.example.com/system \
  --desturi qemu+ssh://dest.example.com/system
```

> **Prerequisites for live migration:**
> - Both hosts must use bridged networking (not NAT)
> - Shared or compatible storage (NFS, SAN, Ceph)
> - Same or compatible CPU models
> - SSH key-based auth between hosts
> - Firewall allows ports 49152-49216 (migration data)

---

## 10. Performance Tuning Tips

### Use Virtio Drivers (Always)

Virtio is paravirtualized — much faster than emulated hardware.

```bash
# In virt-install
--disk ...,bus=virtio          # Virtio block device
--network ...,model=virtio     # Virtio NIC
```

### CPU Pinning (for latency-sensitive workloads)

```bash
# Pin vCPU 0 of VM to physical CPU 2
virsh vcpupin <vmname> 0 2

# Pin all vCPUs (persistent)
virsh vcpupin <vmname> 0 2 --config
virsh vcpupin <vmname> 1 3 --config
```

### Huge Pages

```bash
# On host: allocate huge pages
echo 512 > /proc/sys/vm/nr_hugepages

# In VM XML (virsh edit):
<memoryBacking>
  <hugepages/>
</memoryBacking>
```

### Disk I/O Best Practices

```bash
# For DB/high-I/O VMs
--disk path=...,cache=none,io=native,bus=virtio
```

### Memory Ballooning

Allows the hypervisor to reclaim unused guest memory dynamically.

```xml
<!-- In domain XML -->
<memballoon model='virtio'>
  <stats period='10'/>
</memballoon>
```

Requires `virtio_balloon` kernel module in the guest (present by default in RHEL guests).

---

## 11. Snapshots & Cloning

### Snapshots (qcow2 images only)

```bash
# Create a snapshot
virsh snapshot-create-as <vmname> snap1 "Before upgrade" --disk-only --atomic

# List snapshots
virsh snapshot-list <vmname>

# Revert to snapshot
virsh snapshot-revert <vmname> snap1

# Delete a snapshot
virsh snapshot-delete <vmname> snap1

# Show snapshot info
virsh snapshot-info <vmname> snap1
```

> **Note:** Snapshots of running VMs capture RAM state too (live snapshot). Disk-only snapshots (`--disk-only`) are faster but don't capture RAM state.

### Clone a VM

```bash
# Must be shut down first
virsh shutdown <vmname>

# Clone with virt-clone
virt-clone \
  --original <source-vmname> \
  --name <new-vmname> \
  --file /var/lib/libvirt/images/new-vm.qcow2

# Auto-generate name and file
virt-clone --original <vmname> --auto-clone
```

> `virt-clone` also regenerates MAC addresses and (optionally) SSH host keys if `virt-sysprep` is run afterward.

### Sysprep (Generalize a Clone)

```bash
# Reset machine-specific data (hostname, SSH keys, MAC history, etc.)
virt-sysprep -d <vmname>

# Or on an offline image
virt-sysprep -a /path/to/disk.qcow2
```

---

## 12. Troubleshooting Quick Reference

### VM Won't Start

```bash
virsh start <vmname>          # Check error output
journalctl -u libvirtd        # libvirt logs
cat /var/log/libvirt/qemu/<vmname>.log   # QEMU-level logs
```

### Can't Connect to Guest Console

```bash
# Ensure serial console is configured in guest
# Add to /etc/default/grub on guest:
# GRUB_CMDLINE_LINUX="... console=ttyS0,115200n8"
# Then: grub2-mkconfig -o /boot/grub2/grub.cfg
```

### Network Not Working in Guest

```bash
# Check default network is active
virsh net-list --all
virsh net-start default

# Check iptables/firewalld rules (libvirt adds its own)
iptables -L -n -v | grep virbr

# Restart libvirtd if rules are broken
systemctl restart libvirtd
```

### Storage Pool Issues

```bash
virsh pool-list --all          # List all pools
virsh pool-start default       # Start default pool
virsh pool-refresh default     # Refresh pool state
virsh pool-info default        # Pool details
```

### Check VM Resource Usage from Host

```bash
virt-top                        # Top-like view for VMs (install separately)
virsh domstats <vmname>         # All stats for a VM
```

### Permissions Issue

```bash
# virt-install may need root for some operations
sudo virt-install ...

# Or add user to libvirt group
usermod -aG libvirt $USER
# Re-login for group change to take effect
```

---

## 13. Quick-Reference Cheat Sheet

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 INSTALL METHODS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 ISO        --cdrom /path/to/file.iso
 Import     --import --disk /path/to/disk.qcow2
 Network    --location http://repo/path/to/os
 PXE        --network=bridge:br0 --pxe
 Kickstart  --initrd-inject /ks.cfg --extra-args "ks=file:/ks.cfg"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 NETWORK TYPES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 NAT        --network default
 Bridge     --network br0
 Static     --network br0 --extra-args "ip=IP::GW:MASK:HOST:eth0:none"
 None       --network=none

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 VIRSH ESSENTIALS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 virsh list --all               List all VMs
 virsh start <vm>               Start VM
 virsh shutdown <vm>            Graceful stop
 virsh destroy <vm>             Force stop
 virsh console <vm>             Serial console (Ctrl+] to exit)
 virsh edit <vm>                Edit XML config
 virsh dumpxml <vm>             Show XML config
 virsh snapshot-create-as <vm> <name>   Snapshot
 virsh snapshot-revert <vm> <name>      Revert

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 QEMU-IMG
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 qemu-img create -f qcow2 disk.qcow2 20G
 qemu-img info disk.qcow2
 qemu-img resize disk.qcow2 +10G
 qemu-img convert -f raw -O qcow2 in.raw out.qcow2

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 DEFAULT FILE LOCATIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 VM Images:    /var/lib/libvirt/images/
 VM XML:       /etc/libvirt/qemu/
 QEMU Logs:    /var/log/libvirt/qemu/
 libvirtd log: journalctl -u libvirtd
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

> **References:**
> - RHEL 7 Virtualization Deployment and Administration Guide
> - RHEL 7 Virtualization Security Guide  
> - `man virt-install` · `man virsh` · `man qemu-img`
> - libvirt.org documentation
