# KVM on Ubuntu — Managing Storage
### Engineer's Extensive Reference | Topic: KVM Storage Only
> Ubuntu 20.04 / 22.04 / 24.04 LTS · libvirt · virsh · qemu-img · LVM · NFS · iSCSI  
> Scope: Storage concepts → Storage pools → Storage volumes → Data management → All pool types → Troubleshooting

---

## Table of Contents

1. [Storage Architecture Overview](#1-storage-architecture-overview)
2. [Storage Concepts — Pools and Volumes](#2-storage-concepts--pools-and-volumes)
3. [Preflight — Storage Prerequisites](#3-preflight--storage-prerequisites)
4. [Storage Pool Lifecycle (virsh Fundamentals)](#4-storage-pool-lifecycle-virsh-fundamentals)
5. [Pool Type 1 — Directory-Based](#5-pool-type-1--directory-based)
6. [Pool Type 2 — Disk-Based](#6-pool-type-2--disk-based)
7. [Pool Type 3 — Partition-Based (Filesystem)](#7-pool-type-3--partition-based-filesystem)
8. [Pool Type 4 — LVM-Based](#8-pool-type-4--lvm-based)
9. [Pool Type 5 — NFS-Based](#9-pool-type-5--nfs-based)
10. [Pool Type 6 — iSCSI-Based](#10-pool-type-6--iscsi-based)
11. [Pool Type 7 — GlusterFS-Based](#11-pool-type-7--glusterfs-based)
12. [Pool Type 8 — vHBA / Fibre Channel (SCSI)](#12-pool-type-8--vhba--fibre-channel-scsi)
13. [Managing Storage Volumes](#13-managing-storage-volumes)
14. [Data Management — Wipe, Upload, Download, Resize](#14-data-management--wipe-upload-download-resize)
15. [qemu-img — Disk Image Tool](#15-qemu-img--disk-image-tool)
16. [Adding Storage Devices to Guest VMs](#16-adding-storage-devices-to-guest-vms)
17. [Deleting Storage Pools and Volumes](#17-deleting-storage-pools-and-volumes)
18. [Disk Image Formats Deep Dive](#18-disk-image-formats-deep-dive)
19. [Storage Performance Tuning](#19-storage-performance-tuning)
20. [Monitoring Storage](#20-monitoring-storage)
21. [Troubleshooting Quick Reference](#21-troubleshooting-quick-reference)
22. [Full virsh Storage Command Reference](#22-full-virsh-storage-command-reference)
23. [Quick-Reference Cheat Sheet](#23-quick-reference-cheat-sheet)

---

## 1. Storage Architecture Overview

### 1.1 How KVM Storage Works

```
┌──────────────────────────────────────────────────────────────┐
│                      Ubuntu KVM Host                         │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                     Guest VM                           │  │
│  │   /dev/vda (OS disk)   /dev/vdb (data disk)            │  │
│  └──────────┬──────────────────┬──────────────────────────┘  │
│             │ virtio-blk       │ virtio-scsi                 │
│  ┌──────────▼──────────────────▼──────────────────────────┐  │
│  │               QEMU Block Layer                         │  │
│  │   (emulates block devices, handles image formats)      │  │
│  └──────────┬──────────────────┬──────────────────────────┘  │
│             │                  │                             │
│  ┌──────────▼──────────────────▼──────────────────────────┐  │
│  │                libvirt Storage Layer                   │  │
│  │   Storage Pool A          Storage Pool B               │  │
│  │   (directory)             (LVM / NFS / iSCSI)          │  │
│  │   Volume: vm1.qcow2       Volume: /dev/vg0/lv1         │  │
│  └──────────┬──────────────────┬──────────────────────────┘  │
│             │                  │                             │
│  ┌──────────▼──────────────────▼──────────────────────────┐  │
│  │             Physical Storage Layer                     │  │
│  │   /var/lib/libvirt/images    /dev/sdb    NFS mount     │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### 1.2 The Three-Step Storage Workflow

```
1. CREATE STORAGE POOL   →   2. CREATE STORAGE VOLUME   →   3. ASSIGN TO GUEST VM
   (define where storage        (allocate a piece of            (attach as a block
    lives: dir, LVM, NFS...)     that pool as a disk image)       device to the VM)
```

### 1.3 Key Concepts at a Glance

| Concept | Meaning | Example |
|---|---|---|
| **Storage Pool** | A quantity of storage managed by libvirt | `/var/lib/libvirt/images/` directory |
| **Storage Volume** | A discrete chunk of storage from a pool | `vm1-disk.qcow2` inside the pool |
| **Persistent Pool** | Survives host reboot | Created with `pool-define` |
| **Transient Pool** | Lost after host reboot | Created with `pool-create` |
| **Sparse Volume** | Logical size > physical allocation | qcow2 thin-provisioned image |
| **Thick Volume** | Physical allocation = logical size | `raw` format image |

---

## 2. Storage Concepts — Pools and Volumes

### 2.1 What Is a Storage Pool?

A storage pool is a **libvirt-managed abstraction** over a physical storage resource. Instead of hardcoding raw filesystem paths in VM definitions, you reference pools by name. This means if a path changes (e.g. NFS remount point), you only change the pool — not every VM XML file that uses it.

**Benefits:**
- Centralised storage management through the libvirt API or `virsh`
- Remote management via libvirt's remote protocol (no SSH shell required)
- Queryable capacity/allocation/available stats per pool
- Auto-mount support (NFS, iSCSI) when libvirtd starts

### 2.2 Local vs Networked Storage Pools

| Category | Types | Notes |
|---|---|---|
| **Local** | directory, disk, partition (fs), LVM | Fast; no live migration support |
| **Networked** | NFS, iSCSI, GlusterFS, Fibre Channel | Required for VM live migration |

> **Live migration rule:** Live migration between KVM hosts requires the VM's disk to be on **shared** (networked) storage that both hosts can access simultaneously.

### 2.3 Storage Pool Types Supported on Ubuntu

| Pool Type | virsh type keyword | Backing storage | Ubuntu packages needed |
|---|---|---|---|
| Directory | `dir` | Host filesystem directory | (none — default) |
| Disk | `disk` | Whole physical disk (GPT) | `parted` |
| Filesystem/Partition | `fs` | Formatted partition | `e2fsprogs` |
| LVM Volume Group | `logical` | LVM VG | `lvm2` |
| NFS | `netfs` | NFS export | `nfs-common` |
| iSCSI | `iscsi` | iSCSI target | `open-iscsi` |
| iSCSI direct | `iscsi-direct` | iSCSI target (direct) | `open-iscsi`, `libvirt-daemon-driver-storage-iscsi-direct` |
| GlusterFS | `gluster` | GlusterFS volume | `glusterfs-client` |
| SCSI/vHBA | `scsi` | Fibre Channel HBA | HBA drivers |
| RBD (Ceph) | `rbd` | Ceph cluster | `libvirt-daemon-driver-storage-rbd` |

> **Ubuntu note:** Unlike RHEL, Ubuntu supports RBD (Ceph) pools via `libvirt-daemon-driver-storage-rbd`. Install it separately: `sudo apt install libvirt-daemon-driver-storage-rbd`

### 2.4 Storage Volume Abstraction

A storage volume is presented to a guest as a **local block device** (e.g. `/dev/vda`) regardless of the underlying storage. The guest doesn't know or care whether it's backed by:
- A file on a local directory
- An LVM logical volume
- An iSCSI LUN
- An NFS file share

---

## 3. Preflight — Storage Prerequisites

```bash
# 1. Install core storage packages
sudo apt update && sudo apt install -y \
  qemu-kvm \
  libvirt-daemon-system \
  libvirt-clients \
  virtinst \
  lvm2 \
  parted \
  gdisk \
  e2fsprogs \
  xfsprogs \
  nfs-common \
  open-iscsi \
  libguestfs-tools \
  qemu-utils

# 2. Verify libvirtd is running
sudo systemctl status libvirtd              # Ubuntu 20.04
sudo systemctl status virtqemud             # Ubuntu 22.04+

# 3. Verify default storage pool exists and is active
virsh pool-list --all
# NAME      STATE    AUTOSTART
# default   active   yes        ← Should be here

# 4. Check default pool path
virsh pool-info default
# Shows: Target path = /var/lib/libvirt/images

# 5. Check available disk space on default pool
df -h /var/lib/libvirt/images

# 6. Confirm user is in the libvirt group
groups $USER | grep libvirt
# If not: sudo usermod -aG libvirt $USER && newgrp libvirt
```

---

## 4. Storage Pool Lifecycle (virsh Fundamentals)

### 4.1 The Full Pool Lifecycle

```
pool-define   →   pool-build   →   pool-start   →   (in use)   →   pool-destroy   →   pool-undefine
  (define)         (prepare)        (start/mount)                    (stop/unmount)     (remove defn)
```

| virsh Command | Effect | Survives Reboot? |
|---|---|---|
| `pool-define` | Register pool XML in libvirt (persistent) | Yes |
| `pool-create` | Register AND start pool (transient) | No |
| `pool-build` | Initialise the underlying storage (format, create dirs) | N/A |
| `pool-start` | Start (mount NFS, activate VG, etc.) | Depends on autostart |
| `pool-destroy` | Stop (unmount, deactivate) — data intact | N/A |
| `pool-delete` | Remove storage directory/partition — data gone | N/A |
| `pool-undefine` | Remove pool definition from libvirt | N/A |
| `pool-autostart` | Mark to auto-start when libvirtd starts | Permanent |

### 4.2 Quick Reference — All Pool Management Commands

```bash
virsh pool-list                         # List active pools
virsh pool-list --all                   # List all pools (including inactive)
virsh pool-list --details               # List with capacity info
virsh pool-info <pool>                  # Detailed info: state, capacity, allocation
virsh pool-dumpxml <pool>               # Show full XML definition
virsh pool-define /path/pool.xml        # Define persistent pool from XML
virsh pool-define-as <name> <type> ...  # Define persistent pool from CLI args
virsh pool-create /path/pool.xml        # Define + start transient pool from XML
virsh pool-create-as <name> <type> ...  # Define + start transient pool from CLI
virsh pool-build <pool>                 # Initialise storage (format/create dirs)
virsh pool-start <pool>                 # Start (activate) a defined pool
virsh pool-destroy <pool>              # Stop pool — data safe
virsh pool-delete <pool>               # Delete underlying storage — DATA GONE
virsh pool-undefine <pool>             # Remove pool definition
virsh pool-autostart <pool>            # Enable autostart
virsh pool-autostart --disable <pool>  # Disable autostart
virsh pool-refresh <pool>              # Rescan pool for new volumes
virsh pool-edit <pool>                 # Edit pool XML in $EDITOR
```

### 4.3 Pool State Machine

```
Undefined → pool-define → Defined (inactive)
                               │
                           pool-build (optional, for disk/fs/logical)
                               │
                           pool-start → Active (running)
                               │              │
                           pool-destroy ←─────┘
                               │
                           pool-undefine → Undefined
```

---

## 5. Pool Type 1 — Directory-Based

The simplest and most common pool type. A regular host directory where VM disk image files live.

### 5.1 About Directory Pools

- Default pool (`/var/lib/libvirt/images/`) is a directory pool
- Volumes are files: `.qcow2`, `.raw`, `.img`
- Supports qcow2 (snapshots, COW, compression) and raw formats
- Not suitable for live migration (local storage)

### 5.2 Create via CLI (pool-define-as)

```bash
# Create the host directory
sudo mkdir -p /data/vm-images
sudo chown root:root /data/vm-images
sudo chmod 711 /data/vm-images    # libvirt-qemu needs execute permission

# Define the pool
virsh pool-define-as \
  vm-images \          # Pool name
  dir \               # Pool type
  --target /data/vm-images

# Verify definition
virsh pool-list --all
# vm-images   inactive   no

# Build (creates the directory if needed, sets permissions)
virsh pool-build vm-images

# Start the pool
virsh pool-start vm-images

# Enable autostart
virsh pool-autostart vm-images

# Verify
virsh pool-info vm-images
# Name:           vm-images
# State:          running
# Autostart:      yes
# Capacity:       xxx GB
```

### 5.3 Create via XML

```bash
cat > /tmp/dir-pool.xml <<'EOF'
<pool type='dir'>
  <name>vm-images</name>
  <target>
    <path>/data/vm-images</path>
    <permissions>
      <mode>0711</mode>
      <owner>0</owner>    <!-- root uid -->
      <group>0</group>    <!-- root gid -->
    </permissions>
  </target>
</pool>
EOF

sudo virsh pool-define /tmp/dir-pool.xml
sudo virsh pool-build vm-images
sudo virsh pool-start vm-images
sudo virsh pool-autostart vm-images
```

### 5.4 AppArmor Consideration (Ubuntu-Specific)

If the directory is not under `/var/lib/libvirt/images/`, AppArmor may block QEMU from accessing it:

```bash
# Check if AppArmor is blocking access
sudo aa-status | grep qemu
journalctl -xe | grep apparmor | tail -10

# Fix: add the path to the libvirt-qemu AppArmor profile
sudo nano /etc/apparmor.d/abstractions/libvirt-qemu
# Add line:
#   /data/vm-images/** rwk,

# Reload the profile
sudo apparmor_parser -r /etc/apparmor.d/abstractions/libvirt-qemu
sudo systemctl restart libvirtd
```

---

## 6. Pool Type 2 — Disk-Based

Uses a **whole physical disk** as the pool. libvirt creates partitions as volumes on that disk. The disk must have a GPT partition table.

### 6.1 About Disk Pools

- Whole physical disk is used (e.g. `/dev/sdb`)
- **Never use a disk that has data** — creating the pool may reformat it
- Guests should get partitions (volumes), not the raw disk device
- Volumes appear as: `/dev/sdb1`, `/dev/sdb2`, etc.

### 6.2 Prepare the Disk (GPT Label)

```bash
# Identify target disk
lsblk -d -o NAME,SIZE,TYPE,MODEL
# sdb  500G  disk  Samsung SSD 870

# Verify no mounted partitions
lsblk /dev/sdb

# Create GPT partition table using parted
sudo parted /dev/sdb --script mklabel gpt
# OR interactively:
sudo parted /dev/sdb
# (parted) mklabel gpt
# (parted) quit

# Verify
sudo parted /dev/sdb print
# Partition Table: gpt

# Alternative using gdisk
sudo gdisk /dev/sdb
# o (create GPT), w (write)
```

### 6.3 Create via CLI

```bash
virsh pool-define-as \
  phy-disk \
  disk \
  --source-format gpt \
  --source-dev /dev/sdb \
  --target /dev

virsh pool-build phy-disk     # Writes GPT label if not done manually
virsh pool-start phy-disk
virsh pool-autostart phy-disk
virsh pool-info phy-disk
```

### 6.4 XML Definition

```xml
<pool type='disk'>
  <name>phy-disk</name>
  <source>
    <device path='/dev/sdb'/>
    <format type='gpt'/>
  </source>
  <target>
    <path>/dev</path>         <!-- Volumes appear as /dev/sdb1, /dev/sdb2, etc. -->
  </target>
</pool>
```

---

## 7. Pool Type 3 — Partition-Based (Filesystem)

Uses a **pre-formatted partition** as the pool. libvirt mounts the partition to a target directory and stores volume files there.

### 7.1 About Filesystem Pools

- Pool source: a block partition (e.g. `/dev/sdc1`)
- Target: a mount point directory on the host
- libvirt mounts/unmounts on pool-start/pool-destroy
- Volumes are files stored on the mounted filesystem

### 7.2 Prepare the Partition

```bash
# Create a partition on the disk
sudo fdisk /dev/sdc
# n (new partition), p (primary), 1 (number), enter (default start/end), w (write)

# Or with parted (non-interactive)
sudo parted /dev/sdc --script mklabel gpt
sudo parted /dev/sdc --script mkpart primary 0% 100%

# Format the partition (ext4 recommended for stability)
sudo mkfs.ext4 /dev/sdc1
# Label it for easy identification
sudo e2label /dev/sdc1 kvm-pool

# Create the mount point
sudo mkdir -p /guest_images
```

### 7.3 Create via CLI

```bash
virsh pool-define-as \
  guest-images-fs \
  fs \
  --source-dev /dev/sdc1 \
  --target /guest_images

virsh pool-build guest-images-fs      # Formats if needed; fails if format mismatch
virsh pool-start guest-images-fs
virsh pool-autostart guest-images-fs

# Verify it's mounted
mount | grep /guest_images
# /dev/sdc1 on /guest_images type ext4 (rw,...)

# Verify pool info
virsh pool-info guest-images-fs
# State:     running
# Capacity:  xxx GB
# Allocation: xxx MB
# Available:  xxx GB

# Check for lost+found (confirms mount worked)
ls -la /guest_images
# drwx------  2 root root 16384 ... lost+found
```

### 7.4 XML Definition

```xml
<pool type='fs'>
  <name>guest-images-fs</name>
  <source>
    <device path='/dev/sdc1'/>
    <format type='ext4'/>
  </source>
  <target>
    <path>/guest_images</path>
    <permissions>
      <mode>0711</mode>
      <owner>0</owner>
      <group>0</group>
    </permissions>
  </target>
</pool>
```

---

## 8. Pool Type 4 — LVM-Based

Uses an **LVM Volume Group (VG)** as the pool. Each storage volume is an LVM Logical Volume (LV). Best performance for local storage with fine-grained allocation control.

### 8.1 About LVM Pools

- Volumes are LVM logical volumes (block devices, not files)
- Fast: no filesystem overhead on the volume itself
- Supports thin provisioning (but libvirt doesn't expose all thin features)
- Volume paths: `/dev/VG-name/LV-name`
- **Do NOT run `pool-build`** on an existing VG — it will reformat it

### 8.2 Prepare the LVM VG (If Not Already Existing)

```bash
# Install LVM tools
sudo apt install lvm2

# Step 1: Create a Physical Volume (PV)
sudo pvcreate /dev/sdb
sudo pvcreate /dev/sdc          # Multiple disks can form one VG

# Step 2: Create a Volume Group (VG)
sudo vgcreate libvirt-lvm /dev/sdb
# OR with multiple disks:
sudo vgcreate libvirt-lvm /dev/sdb /dev/sdc

# Step 3: Verify VG
sudo vgs
# VG          #PV  #LV  #SN  Attr   VSize   VFree
# libvirt-lvm   1    0    0  wz--n- 500.00g 500.00g

sudo pvs
sudo vgdisplay libvirt-lvm
```

### 8.3 Create via CLI

```bash
# Using an existing VG (DO NOT run pool-build!)
virsh pool-define-as \
  guest-images-lvm \
  logical \
  --source-dev /dev/sdb \
  --source-name libvirt-lvm \
  --target /dev/libvirt-lvm

virsh pool-start guest-images-lvm
virsh pool-autostart guest-images-lvm

# Verify
virsh pool-info guest-images-lvm
virsh vol-list guest-images-lvm     # Shows existing LVs in the VG
```

### 8.4 Create with a New VG (libvirt manages VG creation)

```bash
# When the VG does NOT exist yet — let libvirt create it
virsh pool-define-as \
  guest-images-lvm \
  logical \
  --source-dev /dev/sdb \
  --source-name libvirt-lvm \
  --target /dev/libvirt-lvm

# pool-build creates the VG (runs pvcreate + vgcreate internally)
virsh pool-build guest-images-lvm

virsh pool-start guest-images-lvm
virsh pool-autostart guest-images-lvm
```

### 8.5 XML Definition

```xml
<pool type='logical'>
  <name>guest-images-lvm</name>
  <source>
    <device path='/dev/sdb'/>
    <!-- For multi-disk VG, add more <device> entries: -->
    <!-- <device path='/dev/sdc'/> -->
    <name>libvirt-lvm</name>
    <format type='lvm2'/>
  </source>
  <target>
    <path>/dev/libvirt-lvm</path>
  </target>
</pool>
```

### 8.6 LVM Thin Provisioning with KVM (Advanced)

```bash
# Create a thin pool LV
sudo lvcreate -L 200G --thinpool thin-pool libvirt-lvm

# Create thin volumes from it (libvirt doesn't manage thin pools directly)
sudo lvcreate -V 20G --thin -n vm1-disk libvirt-lvm/thin-pool
sudo lvcreate -V 20G --thin -n vm2-disk libvirt-lvm/thin-pool

# Refresh the libvirt pool to see the new volumes
virsh pool-refresh guest-images-lvm
virsh vol-list guest-images-lvm
```

---

## 9. Pool Type 5 — NFS-Based

Uses a **remote NFS export** as a storage pool. libvirt mounts the NFS share automatically. Essential for live migration (both hosts can access the same storage).

### 9.1 About NFS Pools

- Pool source: NFS server hostname + exported path
- Target: local mount point on the KVM host
- libvirt mounts on pool-start and unmounts on pool-destroy
- `pool-destroy` = unmount only — data is not deleted
- Volumes are files on the NFS share

### 9.2 Set Up an NFS Server (Ubuntu)

```bash
# On the NFS server
sudo apt install nfs-kernel-server

# Create the export directory
sudo mkdir -p /exports/kvm-storage
sudo chown nobody:nogroup /exports/kvm-storage
sudo chmod 755 /exports/kvm-storage

# Configure exports
sudo nano /etc/exports
# Add:
# /exports/kvm-storage  192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)
# no_root_squash is important for KVM — allows root to write files

# Apply exports
sudo exportfs -ra
sudo systemctl enable --now nfs-kernel-server

# Verify
showmount -e localhost
# Export list for localhost:
# /exports/kvm-storage 192.168.1.0/24
```

### 9.3 Configure the NFS Client (KVM Host)

```bash
# Install NFS client tools
sudo apt install nfs-common

# Test mounting manually first
sudo mkdir -p /mnt/test-nfs
sudo mount -t nfs 192.168.1.50:/exports/kvm-storage /mnt/test-nfs
ls /mnt/test-nfs
sudo umount /mnt/test-nfs
```

### 9.4 Create the NFS Pool via CLI

```bash
virsh pool-define-as \
  nfs-pool \
  netfs \
  --source-host 192.168.1.50 \
  --source-path /exports/kvm-storage \
  --target /var/lib/libvirt/images/nfs-pool

# Create the target mount point if needed
sudo mkdir -p /var/lib/libvirt/images/nfs-pool

virsh pool-build nfs-pool
virsh pool-start nfs-pool
virsh pool-autostart nfs-pool

# Verify NFS is mounted
mount | grep nfs-pool
# 192.168.1.50:/exports/kvm-storage on /var/lib/libvirt/images/nfs-pool type nfs4

virsh pool-info nfs-pool
virsh vol-list nfs-pool
```

### 9.5 XML Definition

```xml
<pool type='netfs'>
  <name>nfs-pool</name>
  <source>
    <host name='192.168.1.50'/>
    <dir path='/exports/kvm-storage'/>
    <format type='auto'/>
  </source>
  <target>
    <path>/var/lib/libvirt/images/nfs-pool</path>
    <permissions>
      <mode>0755</mode>
      <owner>0</owner>
      <group>0</group>
    </permissions>
  </target>
</pool>
```

### 9.6 NFS Pool with NFSv3 (Force Version)

```xml
<pool type='netfs'>
  <name>nfs3-pool</name>
  <source>
    <host name='192.168.1.50'/>
    <dir path='/exports/kvm-storage'/>
    <format type='nfs'/>    <!-- Force NFS protocol detection -->
  </source>
  <target>
    <path>/var/lib/libvirt/images/nfs3-pool</path>
  </target>
</pool>
```

---

## 10. Pool Type 6 — iSCSI-Based

Uses **iSCSI LUNs** as storage volumes. iSCSI provides block-level storage over Ethernet. The iSCSI initiator (KVM host) connects to an iSCSI target (storage server).

### 10.1 About iSCSI Pools

- Volumes are iSCSI LUNs (block devices, not files)
- Volumes appear under `/dev/disk/by-path/`
- Excellent for SAN storage without Fibre Channel infrastructure
- Can be shared between multiple hosts for live migration

### 10.2 Set Up an iSCSI Target (Ubuntu — Using targetcli)

```bash
# On the iSCSI TARGET server
sudo apt install targetcli-fb

# Launch targetcli
sudo targetcli
```

```
# Inside targetcli interactive shell:
# Create a file-backed disk object
/backstores/fileio> create fileio1 /var/lib/iscsi-storage/disk1.img 50G sparse=true

# OR create a block-backed object from a real device
/backstores/block> create block1 /dev/sdb1

# Navigate to iSCSI and create a target
/iscsi> create iqn.2024-01.com.example:storage-server1

# Create a LUN mapping
/iscsi/iqn.2024-01.com.example:storage-server1/tpg1/luns> create /backstores/fileio/fileio1
/iscsi/iqn.2024-01.com.example:storage-server1/tpg1/luns> create /backstores/block/block1

# Set portal (listen on all IPs port 3260)
/iscsi/iqn.2024-01.com.example:storage-server1/tpg1/portals> create 0.0.0.0

# Create ACL (allow any initiator — for testing; restrict in production)
/iscsi/iqn.2024-01.com.example:storage-server1/tpg1/acls> create iqn.2024-01.com.example:kvm-host

# Disable authentication for testing (NOT for production)
/iscsi/iqn.2024-01.com.example:storage-server1/tpg1> set attribute authentication=0

# Save config and exit
saveconfig
exit
```

```bash
# Enable and start the iSCSI target service
sudo systemctl enable --now rtslib-fb-targetctl  # or: target (package-dependent)
# On Ubuntu, check: sudo systemctl list-units | grep iscsi

# Allow iSCSI through UFW
sudo ufw allow 3260/tcp
```

### 10.3 Configure the iSCSI Initiator (KVM Host)

```bash
# Install iSCSI initiator
sudo apt install open-iscsi

# Set a unique initiator IQN
sudo nano /etc/iscsi/initiatorname.iscsi
# InitiatorName=iqn.2024-01.com.example:kvm-host

sudo systemctl enable --now iscsid

# Discover targets on the iSCSI server
sudo iscsiadm --mode discovery --type sendtargets --portal 192.168.1.60
# 192.168.1.60:3260,1 iqn.2024-01.com.example:storage-server1

# Login to the target (attach)
sudo iscsiadm --mode node \
  --targetname iqn.2024-01.com.example:storage-server1 \
  --portal 192.168.1.60:3260 \
  --login

# Verify the iSCSI device appeared
lsblk
# sdb  50G  (the iSCSI-attached disk)

# Logout (detach) — libvirt will manage this after pool setup
sudo iscsiadm --mode node \
  --targetname iqn.2024-01.com.example:storage-server1 \
  --portal 192.168.1.60:3260 \
  --logout
```

### 10.4 Create the iSCSI Pool

```bash
# Define via CLI
virsh pool-define-as \
  iscsi-pool \
  iscsi \
  --source-host 192.168.1.60 \
  --source-dev iqn.2024-01.com.example:storage-server1 \
  --target /dev/disk/by-path

virsh pool-start iscsi-pool
virsh pool-autostart iscsi-pool

# Verify — volumes are iSCSI LUNs
virsh vol-list iscsi-pool
# Lists LUN paths like:
# ip-192.168.1.60:3260-iscsi-iqn...-lun-0
```

### 10.5 XML Definition

```xml
<pool type='iscsi'>
  <name>iscsi-pool</name>
  <source>
    <host name='192.168.1.60'/>
    <device path='iqn.2024-01.com.example:storage-server1'/>
  </source>
  <target>
    <path>/dev/disk/by-path</path>
  </target>
</pool>
```

### 10.6 Secure iSCSI Pool with CHAP Authentication

```bash
# Step 1: Create a libvirt secret for CHAP credentials
cat > /tmp/iscsi-secret.xml <<'EOF'
<secret ephemeral='no' private='yes'>
  <description>iSCSI CHAP credentials for storage-server1</description>
  <usage type='iscsi'>
    <target>iqn.2024-01.com.example:storage-server1</target>
  </usage>
</secret>
EOF

virsh secret-define /tmp/iscsi-secret.xml
# Secret 2d7891af-20be-4e5e-af83-190e8a922360 created

# Step 2: Assign the password to the secret
MYSECRET=$(printf '%s' "your-chap-password" | base64)
virsh secret-set-value 2d7891af-20be-4e5e-af83-190e8a922360 $MYSECRET

# Step 3: Reference the secret in the pool XML
cat > /tmp/iscsi-auth-pool.xml <<'EOF'
<pool type='iscsi'>
  <name>iscsi-auth-pool</name>
  <source>
    <host name='192.168.1.60'/>
    <device path='iqn.2024-01.com.example:storage-server1'/>
    <auth type='chap' username='your-chap-username'>
      <secret uuid='2d7891af-20be-4e5e-af83-190e8a922360'/>
    </auth>
  </source>
  <target>
    <path>/dev/disk/by-path</path>
  </target>
</pool>
EOF

virsh pool-define /tmp/iscsi-auth-pool.xml
virsh pool-start iscsi-auth-pool
virsh pool-autostart iscsi-auth-pool

# List secrets
virsh secret-list
```

---

## 11. Pool Type 7 — GlusterFS-Based

Uses a **GlusterFS distributed filesystem** volume as a storage pool. GlusterFS is a FUSE-based distributed filesystem that scales across multiple hosts.

### 11.1 About GlusterFS Pools

- Pool source: GlusterFS server hostname + volume name
- Useful for distributed, replicated storage across multiple KVM hosts
- Supports VM live migration (if both hosts mount the same Gluster volume)

### 11.2 Install GlusterFS Client on KVM Host

```bash
# Install GlusterFS FUSE client
sudo apt install glusterfs-client

# Verify FUSE is available
ls /dev/fuse
modprobe fuse
```

### 11.3 AppArmor for GlusterFS (Ubuntu-Specific)

On RHEL this used SELinux booleans (`virt_use_fusefs`). On Ubuntu, configure AppArmor:

```bash
# Allow QEMU to access FUSE-mounted filesystems
sudo nano /etc/apparmor.d/abstractions/libvirt-qemu
# Add:
#   /run/gluster/** rwk,
#   /mnt/gluster/** rwk,

sudo apparmor_parser -r /etc/apparmor.d/abstractions/libvirt-qemu
sudo systemctl restart libvirtd
```

### 11.4 Discover GlusterFS Volume Info

```bash
# On the GlusterFS server (or from the client with admin access)
sudo gluster volume status
# Shows: IP address, port, online status, PID

sudo gluster volume info gluster-vol1
# Shows: volume name, type, bricks, options
```

### 11.5 Create the GlusterFS Pool

```bash
virsh pool-define-as \
  gluster-pool \
  gluster \
  --source-host 192.168.1.70 \
  --source-name gluster-vol1 \
  --source-path /

virsh pool-start gluster-pool
virsh pool-autostart gluster-pool
virsh pool-info gluster-pool
virsh vol-list gluster-pool
```

### 11.6 XML Definition

```xml
<pool type='gluster'>
  <name>gluster-pool</name>
  <source>
    <host name='192.168.1.70' port='24007'/>
    <name>gluster-vol1</name>
    <dir path='/'/>
  </source>
</pool>
```

---

## 12. Pool Type 8 — vHBA / Fibre Channel (SCSI)

Uses **Fibre Channel (FC)** storage via N_Port ID Virtualization (NPIV). A physical HBA is virtualised into multiple virtual HBAs (vHBAs), each presented to a separate guest.

### 12.1 About vHBA Pools

- Requires physical Fibre Channel HBA hardware
- NPIV allows multiple vHBAs from one physical HBA
- Each guest gets its own WWNN/WWPN for LUN access
- Best for enterprise SAN environments with FC infrastructure

### 12.2 Verify HBA Supports NPIV

```bash
# List HBAs that support NPIV (vports capability)
virsh nodedev-list --cap vports
# scsi_host3
# scsi_host4

# View HBA details (WWNN, WWPN, max_vports)
virsh nodedev-dumpxml scsi_host3
# Shows: <wwnn>, <wwpn>, <max_vports>, <vports>
```

### 12.3 Create a vHBA

```bash
# Method 1: Define by parent host name
cat > /tmp/vhba-host3.xml <<'EOF'
<device>
  <parent>scsi_host3</parent>
  <capability type='scsi_host'>
    <capability type='fc_host'>
    </capability>
  </capability>
</device>
EOF

virsh nodedev-create /tmp/vhba-host3.xml
# Node device scsi_host5 created from vhba-host3.xml

# Method 2: Define by WWNN/WWPN pair (recommended — survives hardware changes)
cat > /tmp/vhba-wwnn.xml <<'EOF'
<device>
  <name>vhba</name>
  <parent wwnn='20000000c9848140' wwpn='10000000c9848140'/>
  <capability type='scsi_host'>
    <capability type='fc_host'>
    </capability>
  </capability>
</device>
EOF

virsh nodedev-create /tmp/vhba-wwnn.xml

# Verify the new vHBA
virsh nodedev-dumpxml scsi_host5
```

### 12.4 Create the vHBA Storage Pool

```bash
# Single vHBA on an HBA
virsh pool-define-as \
  vhba-pool \
  scsi \
  --adapter-wwnn 5001a4a93526d0a1 \
  --adapter-wwpn 5001a4ace3ee047d \
  --target /dev/disk/by-path

virsh pool-build vhba-pool
virsh pool-start vhba-pool
virsh pool-autostart vhba-pool
```

### 12.5 XML Definitions

```xml
<!-- Single pool — only pool using this HBA -->
<pool type='scsi'>
  <name>vhba-pool-single</name>
  <source>
    <adapter type='fc_host'
             wwnn='5001a4a93526d0a1'
             wwpn='5001a4ace3ee047d'/>
  </source>
  <target>
    <path>/dev/disk/by-path</path>
  </target>
</pool>

<!-- Multiple pools on same HBA — use parent attribute -->
<pool type='scsi'>
  <name>vhba-pool-multi</name>
  <source>
    <adapter type='fc_host'
             parent='scsi_host3'
             wwnn='5001a4a93526d0a1'
             wwpn='5001a4ace3ee047d'/>
  </source>
  <target>
    <path>/dev</path>    <!-- Use /dev/ when multiple pools share the same HBA -->
  </target>
</pool>
```

> **Ubuntu path note:** When multiple vHBAs share one physical HBA, set `<path>/dev/</path>`. When only one pool exists on the HBA, use `/dev/disk/by-path` for stable persistent paths.

---

## 13. Managing Storage Volumes

### 13.1 Volume Concepts

A storage volume is a discrete allocation from a storage pool — the actual "disk" seen by a guest VM.

| Pool type | Volume type | Example path |
|---|---|---|
| dir / fs / NFS | File (qcow2, raw) | `/var/lib/libvirt/images/vm1.qcow2` |
| LVM | Logical Volume | `/dev/libvirt-lvm/vm1-disk` |
| disk | Disk partition | `/dev/sdb1` |
| iSCSI / FC | LUN (block device) | `/dev/disk/by-path/ip-...` |

### 13.2 Create a Volume via XML

```bash
cat > /tmp/volume1.xml <<'EOF'
<volume>
  <name>vm1-disk.qcow2</name>
  <capacity unit='G'>20</capacity>      <!-- Logical size -->
  <allocation unit='G'>0</allocation>   <!-- Physical pre-allocation (0 = sparse) -->
  <target>
    <format type='qcow2'/>
    <permissions>
      <mode>0600</mode>
      <owner>64055</owner>              <!-- libvirt-qemu uid: id libvirt-qemu -->
      <group>108</group>                <!-- kvm gid: getent group kvm -->
    </permissions>
  </target>
</volume>
EOF

virsh vol-create guest-images-dir /tmp/volume1.xml
# Vol vm1-disk.qcow2 created
```

### 13.3 Create a Volume via CLI (vol-create-as)

```bash
# Basic: creates a sparse qcow2 in the specified pool
virsh vol-create-as \
  guest-images-dir \      # Pool name
  vm1-disk.qcow2 \        # Volume name
  20G \                   # Capacity (logical size)
  --allocation 0 \        # Physical allocation (0 = thin/sparse)
  --format qcow2          # Format: qcow2 | raw | vmdk

# Create a pre-allocated raw volume
virsh vol-create-as \
  guest-images-dir \
  vm2-raw-disk \
  20G \
  --allocation 20G \      # Fully pre-allocate
  --format raw

# Create in an LVM pool (logical volume)
virsh vol-create-as \
  guest-images-lvm \
  vm3-lv \
  30G

# Create a disk-pool partition
virsh vol-create-as \
  phy-disk \
  partition1 \
  10G
```

### 13.4 Clone a Volume

```bash
# Clone within the same pool
virsh vol-clone \
  --pool guest-images-dir \
  vm1-disk.qcow2 \          # Source volume
  vm1-clone.qcow2           # New volume name

# Clone to a different pool
virsh vol-clone \
  --pool guest-images-dir \
  vm1-disk.qcow2 \
  --newname vm1-nfs-copy \
  --newpool nfs-pool

# Monitor clone progress (for large volumes)
watch -n2 'ls -lh /var/lib/libvirt/images/vm1-clone.qcow2'
```

### 13.5 List and Inspect Volumes

```bash
# List all volumes in a pool
virsh vol-list guest-images-dir
# Name              Path
# vm1-disk.qcow2    /var/lib/libvirt/images/vm1-disk.qcow2

# Detailed volume listing
virsh vol-list guest-images-dir --details
# Name              Path             Type   Capacity   Allocation
# vm1-disk.qcow2    /path/...        file   20.00 GiB  196.00 KiB

# Get info for a specific volume
virsh vol-info --pool guest-images-dir vm1-disk.qcow2
# Name:         vm1-disk.qcow2
# Type:         file
# Capacity:     20.00 GiB
# Allocation:   196.00 KiB    ← sparse: physical use << logical size

# Get the path of a volume
virsh vol-path --pool guest-images-dir vm1-disk.qcow2
# /var/lib/libvirt/images/vm1-disk.qcow2

# Get volume XML
virsh vol-dumpxml --pool guest-images-dir vm1-disk.qcow2
```

---

## 14. Data Management — Wipe, Upload, Download, Resize

### 14.1 Wipe a Storage Volume

Securely overwrite a volume so data cannot be recovered. Use before deallocating or repurposing a volume.

```bash
# Default wipe: overwrite with zeroes
virsh vol-wipe --pool guest-images-dir vm1-disk.qcow2

# Wipe with a specific algorithm
virsh vol-wipe --pool guest-images-dir vm1-disk.qcow2 --algorithm nnsa

# Available algorithms:
# zero         → Write zeroes (fast, basic)
# nnsa         → 3-pass (DoD 5220.22-M standard)
# dod          → 7-pass DoD standard
# bsi          → 9-pass German BSI standard
# gutmann      → 35-pass Gutmann method
# pfitzner7    → 7-pass Pfitzner
# pfitzner33   → 33-pass Pfitzner
# random       → Write random bytes
```

> **Note:** `vol-wipe` does not work on raw block devices (iSCSI/FC LUNs). Use `dd` or `blkdiscard` directly for those.

### 14.2 Upload Data to a Volume

Copy a local file's contents into a storage volume:

```bash
# Upload a file into a volume (writes from the beginning)
virsh vol-upload \
  --pool disk-pool \
  sde1 \                          # Volume name or path
  /tmp/data-to-upload.img         # Local source file

# Upload with offset (write starting at byte 1048576)
virsh vol-upload \
  --pool disk-pool \
  sde1 \
  /tmp/data.img \
  --offset 1048576

# Upload with length limit (write max 500MB)
virsh vol-upload \
  --pool disk-pool \
  sde1 \
  /tmp/data.img \
  --length 524288000

# Practical use: upload an OS image into a block volume
virsh vol-upload \
  --pool guest-images-lvm \
  vm1-lv \
  /var/lib/libvirt/images/ubuntu-22.04.raw
```

> **Warning:** The local file must not exceed the `--length` limit. Volume must be large enough to hold the data.

### 14.3 Download Data from a Volume

Copy a volume's contents to a local file:

```bash
# Download the entire volume
virsh vol-download \
  --pool disk-pool \
  sde1 \                       # Volume name
  /tmp/volume-backup.img       # Local destination file

# Download with offset and length
virsh vol-download \
  --pool disk-pool \
  sde1 \
  /tmp/partial.img \
  --offset 1048576 \
  --length 524288000

# Practical use: backup a VM disk to a file
POOL="default"
VOL="vm1-disk.qcow2"
BACKUP_PATH="/backups/$(date +%Y%m%d)-${VOL}"

virsh vol-download \
  --pool "$POOL" \
  "$VOL" \
  "$BACKUP_PATH"

echo "Backup complete: $BACKUP_PATH"
ls -lh "$BACKUP_PATH"
```

### 14.4 Resize a Storage Volume

```bash
# Grow a volume (safe for offline volumes)
virsh vol-resize \
  --pool guest-images-dir \
  vm1-disk.qcow2 \
  30G                           # New total size (not increment)

# Grow by a delta (add 10G to current size)
virsh vol-resize \
  --pool guest-images-dir \
  vm1-disk.qcow2 \
  10G \
  --delta                       # Treat capacity as increment

# Shrink a volume (DANGEROUS — data may be lost; rarely safe)
virsh vol-resize \
  --pool guest-images-dir \
  vm1-disk.qcow2 \
  15G \
  --shrink                      # Explicit --shrink required; no negative values

# Pre-allocate space after resize (not sparse anymore)
virsh vol-resize \
  --pool guest-images-dir \
  vm1-disk.qcow2 \
  30G \
  --allocate                    # Forces physical allocation of the new space

# IMPORTANT: After resizing the volume, you must also resize inside the guest:
# See Section 16.5 for the full online disk resize workflow
```

> **Safety rule:** Only resize volumes that are NOT currently in use by a running VM. For live resize, use `virsh blockresize` (see Section 16.5).

---

## 15. qemu-img — Disk Image Tool

`qemu-img` is the primary tool for creating, converting, inspecting, and managing disk image files outside of libvirt.

### 15.1 Install

```bash
sudo apt install qemu-utils
qemu-img --version
```

### 15.2 Create Images

```bash
# Thin-provisioned qcow2 (default for Ubuntu KVM)
qemu-img create -f qcow2 /var/lib/libvirt/images/vm1.qcow2 20G

# Pre-allocated raw image (full 20G allocated immediately)
qemu-img create -f raw /var/lib/libvirt/images/vm1.raw 20G

# qcow2 with backing file (COW overlay — only changed blocks stored)
qemu-img create -f qcow2 \
  -b /var/lib/libvirt/images/base/ubuntu22-base.qcow2 \
  -F qcow2 \
  /var/lib/libvirt/images/overlay-vm1.qcow2 \
  20G

# qcow2 with compression
qemu-img create -f qcow2 -o preallocation=metadata \
  /var/lib/libvirt/images/vm1-meta.qcow2 20G
```

### 15.3 Inspect Images

```bash
# Basic info
qemu-img info /var/lib/libvirt/images/vm1.qcow2
# image: vm1.qcow2
# file format: qcow2
# virtual size: 20 GiB (21474836480 bytes)   ← logical size
# disk size: 2.18 MiB                         ← actual space used
# cluster_size: 65536
# Format specific information:
#   compat: 1.1
#   compression type: zlib
#   lazy refcounts: false
#   refcount bits: 16
#   corrupt: false
#   extended l2: false

# Check backing file chain (for COW overlays)
qemu-img info --backing-chain /var/lib/libvirt/images/overlay-vm1.qcow2

# Check image for errors
qemu-img check /var/lib/libvirt/images/vm1.qcow2
# No errors were found on the image.

# Check with repair
qemu-img check -r all /var/lib/libvirt/images/vm1.qcow2
```

### 15.4 Convert Images

```bash
# Convert qcow2 → raw (full size, no compression)
qemu-img convert \
  -f qcow2 -O raw \
  /var/lib/libvirt/images/vm1.qcow2 \
  /var/lib/libvirt/images/vm1.raw

# Convert raw → qcow2 (compress, thin-provision)
qemu-img convert \
  -f raw -O qcow2 \
  /var/lib/libvirt/images/vm1.raw \
  /var/lib/libvirt/images/vm1.qcow2

# Convert with compression (smaller output file)
qemu-img convert \
  -f qcow2 -O qcow2 \
  -c \                         # Enable compression
  /var/lib/libvirt/images/vm1.qcow2 \
  /var/lib/libvirt/images/vm1-compressed.qcow2

# Convert and show progress
qemu-img convert \
  -p \                         # Show progress bar
  -f qcow2 -O qcow2 \
  input.qcow2 \
  output.qcow2

# Flatten a COW overlay into a standalone image
qemu-img convert \
  -f qcow2 -O qcow2 \
  /var/lib/libvirt/images/overlay-vm1.qcow2 \
  /var/lib/libvirt/images/standalone-vm1.qcow2
```

### 15.5 Resize Images

```bash
# Grow a qcow2 image
qemu-img resize /var/lib/libvirt/images/vm1.qcow2 30G

# Grow by delta
qemu-img resize /var/lib/libvirt/images/vm1.qcow2 +10G

# Shrink (unsafe — must be done carefully with filesystem-aware tools)
qemu-img resize --shrink /var/lib/libvirt/images/vm1.qcow2 15G

# Note: resizing the image file does NOT automatically resize
# the partition or filesystem inside the guest.
# See Section 16.5 for the complete online disk resize workflow.
```

### 15.6 Snapshot Management (qcow2 internal snapshots)

```bash
# List snapshots inside a qcow2 image
qemu-img snapshot -l /var/lib/libvirt/images/vm1.qcow2

# Create an internal snapshot (VM must be stopped, or use virsh)
qemu-img snapshot -c snap1 /var/lib/libvirt/images/vm1.qcow2

# Revert to a snapshot
qemu-img snapshot -a snap1 /var/lib/libvirt/images/vm1.qcow2

# Delete a snapshot
qemu-img snapshot -d snap1 /var/lib/libvirt/images/vm1.qcow2
```

### 15.7 Commit and Rebase (COW Operations)

```bash
# Commit: merge overlay changes back into the base (base absorbs overlay)
# — Run while the overlay VM is stopped
qemu-img commit /var/lib/libvirt/images/overlay-vm1.qcow2

# Rebase: change the backing file of an overlay
qemu-img rebase \
  -b /var/lib/libvirt/images/new-base.qcow2 \
  -F qcow2 \
  /var/lib/libvirt/images/overlay-vm1.qcow2

# Safe rebase (copies all data — no dependency on old base after this)
qemu-img rebase \
  -p \                    # Show progress
  -b "" \                 # No backing file = standalone
  -F qcow2 \
  /var/lib/libvirt/images/overlay-vm1.qcow2
```

### 15.8 Image Format Reference

| Format | Option string | Thin? | Snapshots? | Backing file? | Notes |
|---|---|---|---|---|---|
| qcow2 | `-f qcow2` | Yes | Yes | Yes | Default Ubuntu KVM format |
| raw | `-f raw` | No (unless sparse) | No | No | Best performance |
| vmdk | `-f vmdk` | Yes | Limited | No | VMware interop |
| vdi | `-f vdi` | Yes | Yes | No | VirtualBox interop |
| vhd/vpc | `-f vpc` | Yes | No | No | Hyper-V interop |

---

## 16. Adding Storage Devices to Guest VMs

### 16.1 Attach a Volume at VM Creation (virt-install)

```bash
# Simple: create a new 20GB disk in the default pool
virt-install \
  --name vm1 \
  --memory 2048 --vcpus 2 \
  --disk size=20,format=qcow2,bus=virtio \
  --cdrom /path/to/ubuntu.iso \
  --os-variant ubuntu22.04

# Use a specific volume from a named pool
virt-install \
  --name vm1 \
  --memory 2048 --vcpus 2 \
  --disk vol=guest-images-dir/vm1-disk.qcow2,bus=virtio \
  --import \
  --os-variant ubuntu22.04

# Use an LVM volume
virt-install \
  --name vm1 \
  --memory 2048 --vcpus 2 \
  --disk /dev/libvirt-lvm/vm1-lv,bus=virtio,cache=none,io=native \
  --import \
  --os-variant ubuntu22.04

# Multiple disks (OS disk + data disk)
virt-install \
  --name vm1 \
  --memory 4096 --vcpus 2 \
  --disk vol=default/vm1-os.qcow2,bus=virtio \
  --disk vol=default/vm1-data.qcow2,bus=virtio \
  --import \
  --os-variant ubuntu22.04
```

### 16.2 Attach a Disk to a Running VM (Hot-Plug)

```bash
# Create a volume for the new disk
virsh vol-create-as \
  guest-images-dir \
  vm1-extra-disk.qcow2 \
  50G \
  --format qcow2

# Get the path
VOL_PATH=$(virsh vol-path --pool guest-images-dir vm1-extra-disk.qcow2)

# Hot-attach to a running VM
virsh attach-disk vm1 \
  "$VOL_PATH" \              # Host-side disk path
  vdb \                      # Guest-side device name
  --driver qemu \
  --subdriver qcow2 \
  --cache none \
  --live \                   # Apply immediately (live)
  --persistent               # Persist across reboots

# Verify inside the guest
virsh console vm1
# lsblk   → should show /dev/vdb
```

### 16.3 Disk XML for Domain Definition

```xml
<!-- In guest domain XML (virsh edit <vmname>) -->

<!-- qcow2 image file on a directory pool -->
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none' io='native'/>
  <source file='/var/lib/libvirt/images/vm1-disk.qcow2'/>
  <target dev='vda' bus='virtio'/>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
</disk>

<!-- LVM logical volume -->
<disk type='block' device='disk'>
  <driver name='qemu' type='raw' cache='none' io='native'/>
  <source dev='/dev/libvirt-lvm/vm1-lv'/>
  <target dev='vda' bus='virtio'/>
</disk>

<!-- iSCSI LUN -->
<disk type='block' device='disk'>
  <driver name='qemu' type='raw' cache='none' io='native'/>
  <source dev='/dev/disk/by-path/ip-192.168.1.60:3260-iscsi-iqn.2024...-lun-0'/>
  <target dev='vda' bus='virtio'/>
</disk>

<!-- Using storage pool + volume name (recommended — path-agnostic) -->
<disk type='volume' device='disk'>
  <driver name='qemu' type='qcow2' cache='none'/>
  <source pool='guest-images-dir' volume='vm1-disk.qcow2'/>
  <target dev='vda' bus='virtio'/>
</disk>
```

### 16.4 Detach a Disk from a Running VM

```bash
# Detach by target device name
virsh detach-disk vm1 vdb --live --persistent

# Detach using the XML (more reliable for complex setups)
cat > /tmp/detach-disk.xml <<'EOF'
<disk type='file' device='disk'>
  <source file='/var/lib/libvirt/images/vm1-extra-disk.qcow2'/>
  <target dev='vdb'/>
</disk>
EOF
virsh detach-device vm1 /tmp/detach-disk.xml --live --persistent
```

### 16.5 Grow a Disk Live (Online Resize Workflow)

Complete workflow to grow a VM's disk without rebooting:

```bash
# STEP 1: Grow the volume on the HOST side
# Option A: Grow via virsh (for pool-managed volumes)
virsh vol-resize --pool guest-images-dir vm1-disk.qcow2 30G

# Option B: Grow directly with qemu-img (for image files)
sudo qemu-img resize /var/lib/libvirt/images/vm1-disk.qcow2 +10G

# STEP 2: Notify the running guest of the new size
virsh blockresize vm1 /var/lib/libvirt/images/vm1-disk.qcow2 30G

# STEP 3: Inside the guest — resize the partition
# (requires cloud-guest-utils for growpart)
sudo apt install cloud-guest-utils

# Check current partition layout
lsblk
sudo fdisk -l /dev/vda

# Resize partition 1 to use all available space
sudo growpart /dev/vda 1

# STEP 4: Inside the guest — resize the filesystem
# For ext4:
sudo resize2fs /dev/vda1

# For xfs:
sudo xfs_growfs /

# For LVM inside the VM:
sudo pvresize /dev/vda1
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv   # or xfs_growfs

# Verify
df -h /
# Should show new size
lsblk
```

---

## 17. Deleting Storage Pools and Volumes

### 17.1 Safe Pool Deletion Workflow

```bash
# STEP 1: List all pools and identify the target
virsh pool-list --all

# STEP 2: Ensure no running VMs are using volumes from this pool
# Find VMs using the pool
for VM in $(virsh list --name); do
  virsh domblklist $VM | grep <pool-path> && echo "$VM uses this pool"
done

# STEP 3: Stop the pool (unmounts NFS, deactivates LVM, etc.)
# Data is NOT deleted by pool-destroy
virsh pool-destroy guest-images-pool

# STEP 4 (optional): Delete the underlying storage
# WARNING: THIS DELETES DATA
virsh pool-delete guest-images-pool   # Deletes the target directory/data
# Only use this if you WANT to erase the storage

# STEP 5: Remove the pool definition from libvirt
virsh pool-undefine guest-images-pool

# STEP 6: Confirm
virsh pool-list --all
# Pool should no longer appear
```

### 17.2 Delete a Storage Volume

```bash
# STEP 1: Ensure the volume is not attached to any VM
virsh vol-info --pool guest-images-dir vm1-disk.qcow2

# Check which VMs use this volume
for VM in $(virsh list --all --name); do
  virsh domblklist $VM | grep vm1-disk.qcow2 && echo "$VM uses this volume"
done

# STEP 2: Wipe the volume (optional — secure erase before deletion)
virsh vol-wipe --pool guest-images-dir vm1-disk.qcow2

# STEP 3: Delete the volume
virsh vol-delete vm1-disk.qcow2 --pool guest-images-dir
# Vol vm1-disk.qcow2 deleted

# STEP 4: Verify
virsh vol-list guest-images-dir
# vm1-disk.qcow2 should no longer appear
```

---

## 18. Disk Image Formats Deep Dive

### 18.1 qcow2 (Recommended for Ubuntu KVM)

**QEMU Copy-on-Write version 2**

| Feature | Detail |
|---|---|
| Thin provisioning | Yes — only allocates blocks as written |
| Snapshots | Internal snapshots + external (disk-only) snapshots |
| Backing files | Yes — COW overlays |
| Compression | Optional per-cluster compression |
| Encryption | AES-CBC or LUKS |
| Performance | Slight overhead vs raw (negligible on modern SSDs) |
| Best for | General purpose — dev, test, production |

```bash
# qcow2 advanced options
qemu-img create \
  -f qcow2 \
  -o cluster_size=2M,preallocation=metadata \    # Larger cluster for big sequential writes
  /var/lib/libvirt/images/vm1.qcow2 \
  50G

# Available qcow2 creation options
qemu-img create -f qcow2 -o help
```

### 18.2 raw

**No metadata, bare blocks**

| Feature | Detail |
|---|---|
| Thin provisioning | Only if sparse (sparse file or `fallocate`) |
| Snapshots | No internal snapshots |
| Backing files | No |
| Performance | Best — no metadata overhead |
| Best for | Production databases, high-IOPS workloads, LVM LVs |

```bash
# Create a sparse raw file (thin, grows as written — like qcow2 but no features)
qemu-img create -f raw /var/lib/libvirt/images/vm1.raw 20G

# Create a fully allocated raw file (pre-allocate ALL blocks)
fallocate -l 20G /var/lib/libvirt/images/vm1-full.raw
# or
dd if=/dev/zero of=/var/lib/libvirt/images/vm1-full.raw bs=1M count=20480
```

### 18.3 Choosing the Right Format

```
Is the VM a production database?
  YES → raw + LVM LV + cache=none + io=native   (best possible I/O)
  NO  → qcow2                                    (flexibility wins)

Do you need snapshots or COW clones?
  YES → qcow2 mandatory

Do you need live migration?
  YES → shared storage (NFS/iSCSI) + either format

Is disk I/O the bottleneck?
  YES → Consider raw format + virtio-scsi bus

Is this a dev/test VM?
  YES → qcow2 + cache=writeback   (faster, acceptable risk)
```

---

## 19. Storage Performance Tuning

### 19.1 Disk Bus Selection

| Bus | Guest device | Performance | Notes |
|---|---|---|---|
| `virtio` | `/dev/vda`, `/dev/vdb` | Excellent | Always use for Ubuntu guests |
| `virtio-scsi` | `/dev/sda`, `/dev/sdb` | Excellent | Supports more devices, SCSI commands |
| `ide` | `/dev/hda` | Poor | Legacy — avoid |
| `sata` | `/dev/sda` | OK | For Windows/legacy |

```bash
# virtio bus (default recommendation)
--disk ...,bus=virtio

# virtio-scsi bus (required for SCSI pass-through)
--disk ...,bus=scsi
```

### 19.2 Cache Mode Selection

| Cache | Host page cache | Flush requests | Best for |
|---|---|---|---|
| `none` | Bypassed (O_DIRECT) | Honoured | Production (databases, critical data) |
| `writeback` | Used | May be ignored | Dev/test (fastest, data-loss risk) |
| `writethrough` | Write-through | Honoured | Safe, slower |
| `directsync` | Write-direct | Honoured | Stricter than none |
| `unsafe` | Full caching | Ignored | CI/benchmarks (fastest, dangerous) |

```bash
# Production setting (avoids double-buffering)
--disk ...,cache=none,io=native

# Dev/test (maximum speed, acceptable risk)
--disk ...,cache=writeback

# In domain XML
<driver name='qemu' type='qcow2' cache='none' io='native'/>
```

### 19.3 I/O Mode

```xml
<!-- native: O_DIRECT — bypasses host page cache entirely -->
<driver name='qemu' type='qcow2' cache='none' io='native'/>

<!-- threads: uses QEMU thread pool (default) -->
<driver name='qemu' type='qcow2' io='threads'/>
```

### 19.4 virtio-scsi vs virtio-blk

```xml
<!-- virtio-blk: simpler, slightly lower latency -->
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none'/>
  <source file='/path/to/disk.qcow2'/>
  <target dev='vda' bus='virtio'/>
</disk>

<!-- virtio-scsi: more devices (up to 256), SCSI commands, discard support -->
<!-- Requires virtio-scsi controller in the domain XML -->
<controller type='scsi' model='virtio-scsi'/>
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none' discard='unmap'/>
  <source file='/path/to/disk.qcow2'/>
  <target dev='sda' bus='scsi'/>
</disk>
```

### 19.5 Enable Discard (TRIM) for qcow2

Allows the guest to release freed blocks back to the host filesystem:

```xml
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none' discard='unmap'/>
  <source file='/path/to/disk.qcow2'/>
  <target dev='sda' bus='scsi'/>
</disk>
```

```bash
# Inside the guest — trigger TRIM/discard
sudo fstrim -v /
# /: 12.5 GiB (13421772800 bytes) trimmed

# Schedule periodic TRIM
sudo systemctl enable fstrim.timer
```

### 19.6 Pre-allocate qcow2 Metadata

Reduces latency from metadata cluster allocation during write bursts:

```bash
# Create qcow2 with metadata pre-allocated (not data — still thin)
qemu-img create -f qcow2 \
  -o preallocation=metadata \
  /var/lib/libvirt/images/vm1-premeta.qcow2 \
  20G

# Full pre-allocation (like raw, but in qcow2 format)
qemu-img create -f qcow2 \
  -o preallocation=full \
  /var/lib/libvirt/images/vm1-full.qcow2 \
  20G
```

---

## 20. Monitoring Storage

### 20.1 Pool and Volume Statistics

```bash
# Pool space summary
virsh pool-info <pool>
# Name:           default
# State:          running
# Capacity:       476.91 GiB
# Allocation:     148.52 GiB
# Available:      328.39 GiB

# All pools with details
virsh pool-list --details
# Name     State    Autostart  Persistent  Capacity    Allocation  Available
# default  running  yes        yes         476.91 GiB  148.52 GiB  328.39 GiB

# Volume physical vs logical size (spot sparse volumes)
virsh vol-info --pool default vm1.qcow2
# Type:         file
# Capacity:     20.00 GiB    ← logical size
# Allocation:   3.12 GiB     ← actual disk usage (sparse!)
```

### 20.2 Per-VM Disk I/O Statistics

```bash
# Block device stats for a running VM
virsh domblkstat vm1 vda
# vda rd_req    5432        ← read requests
# vda rd_bytes  89128960    ← bytes read
# vda wr_req    1203        ← write requests
# vda wr_bytes  12452864    ← bytes written
# vda flush_operations 89

# All block devices
virsh domblklist vm1 --details
virsh domblkstat vm1 --human    # Human-readable sizes

# Disk block info (current usage by a guest)
virsh domblkinfo vm1 vda
# Capacity:      21474836480   ← logical size (bytes)
# Allocation:    3354263552    ← host-side physical allocation
# Physical:      21474836480   ← physical block size reported to guest
```

### 20.3 Storage I/O Throttling (QoS)

```xml
<!-- In guest domain XML — limit disk I/O per device -->
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none'/>
  <source file='/path/to/disk.qcow2'/>
  <target dev='vda' bus='virtio'/>
  <iotune>
    <total_bytes_sec>104857600</total_bytes_sec>       <!-- 100 MB/s total -->
    <read_bytes_sec>52428800</read_bytes_sec>           <!-- 50 MB/s read -->
    <write_bytes_sec>52428800</write_bytes_sec>         <!-- 50 MB/s write -->
    <total_iops_sec>1000</total_iops_sec>               <!-- 1000 IOPS total -->
    <read_iops_sec>500</read_iops_sec>
    <write_iops_sec>500</write_iops_sec>
  </iotune>
</disk>
```

```bash
# Apply I/O throttle live without reboot
virsh blkdeviotune vm1 vda \
  --total-bytes-sec 104857600 \
  --total-iops-sec 1000 \
  --live --config
```

### 20.4 Monitor Disk Usage Trends

```bash
# Watch all image sizes in real time
watch -n5 'du -sh /var/lib/libvirt/images/* | sort -h'

# Check thin provisioning overcommit
for IMG in /var/lib/libvirt/images/*.qcow2; do
  VSIZE=$(qemu-img info "$IMG" | awk '/virtual size/{print $3, $4}')
  DSIZE=$(qemu-img info "$IMG" | awk '/disk size/{print $3, $4}')
  echo "$(basename $IMG): Virtual=$VSIZE, Disk=$DSIZE"
done

# Quick host storage health check
df -h /var/lib/libvirt/images
# Filesystem      Size  Used  Avail  Use%  Mounted on
# /dev/sda1       500G  148G   328G   32%  /
```

---

## 21. Troubleshooting Quick Reference

### "Error: cannot access storage file" / AppArmor Denial

```bash
# Check AppArmor denials
journalctl -xe | grep apparmor | tail -20
sudo aa-status | grep qemu

# Fix: add path to AppArmor profile
sudo nano /etc/apparmor.d/abstractions/libvirt-qemu
# Add:
#   /data/vm-images/** rwk,
#   /mnt/nfs-pool/** rwk,

sudo apparmor_parser -r /etc/apparmor.d/abstractions/libvirt-qemu
sudo systemctl restart libvirtd
```

### Pool Won't Start

```bash
virsh pool-start <pool>
# Check error message carefully

# Debug: check pool XML
virsh pool-dumpxml <pool>

# For NFS pool: verify mount works manually
sudo mount -t nfs 192.168.1.50:/exports/kvm /mnt/test
ls /mnt/test
sudo umount /mnt/test

# For LVM pool: check VG is available
sudo vgs
sudo vgchange -ay libvirt-lvm    # Activate VG manually

# For iSCSI pool: check initiator can discover targets
sudo iscsiadm --mode discovery --type sendtargets --portal 192.168.1.60

# Check libvirtd logs
journalctl -u libvirtd --since "10 min ago"
```

### Volume Not Visible After Creation

```bash
# Refresh the pool (rescan for new volumes)
virsh pool-refresh <pool>

# Then list volumes again
virsh vol-list <pool>
```

### "Pool build failed" for Disk/FS Pool

```bash
# Cause: source device has a different format than expected

# Check current format
sudo parted /dev/sdb print
sudo blkid /dev/sdb

# Force build (overwrites existing data — BACKUP FIRST!)
virsh pool-build <pool> --overwrite
```

### qemu-img Shows Backing File Errors

```bash
# Check the backing chain
qemu-img info --backing-chain /path/to/overlay.qcow2

# Error: "Could not open backing file"
# Cause: backing file moved or deleted
# Fix option 1: repair backing file path
qemu-img rebase -u -b /new/path/to/base.qcow2 /path/to/overlay.qcow2

# Fix option 2: flatten overlay (no longer depends on backing)
qemu-img convert -f qcow2 -O qcow2 overlay.qcow2 standalone.qcow2
```

### "No space left on device" Despite Thin Provisioning

```bash
# Thin-provisioned images CAN fill the host filesystem silently

# Check real physical usage of all qcow2 files
du -sh /var/lib/libvirt/images/*.qcow2 | sort -h

# Check how much the pool has available
virsh pool-info default | grep Available

# Compact a qcow2 (free unused blocks — only if sparse)
# VM must be stopped
virsh shutdown <vm>
qemu-img convert -O qcow2 bloated.qcow2 compact.qcow2
```

### iSCSI LUN Not Appearing in Pool

```bash
# Check iscsid is running
sudo systemctl status iscsid

# Check the target is reachable
sudo iscsiadm --mode discovery --type sendtargets --portal <server-ip>

# Check iSCSI sessions
sudo iscsiadm --mode session

# Refresh the pool
virsh pool-refresh iscsi-pool
virsh vol-list iscsi-pool

# Check /dev/disk/by-path
ls -la /dev/disk/by-path/ | grep iscsi
```

### LVM Volumes Not Visible After Host Reboot

```bash
# VG may not be active
sudo vgs
sudo vgchange -ay libvirt-lvm    # Activate all LVs in VG

# Check LVM service
sudo systemctl status lvm2-monitor

# Restart the pool
virsh pool-start guest-images-lvm
virsh pool-refresh guest-images-lvm
```

---

## 22. Full virsh Storage Command Reference

### Storage Pool Commands

| Command | Usage |
|---|---|
| `pool-list` | `virsh pool-list [--all] [--details]` |
| `pool-info` | `virsh pool-info <pool>` |
| `pool-dumpxml` | `virsh pool-dumpxml <pool>` |
| `pool-define` | `virsh pool-define /path/pool.xml` |
| `pool-define-as` | `virsh pool-define-as <name> <type> [options] --target <path>` |
| `pool-create` | `virsh pool-create /path/pool.xml` |
| `pool-create-as` | `virsh pool-create-as <name> <type> [options] --target <path>` |
| `pool-build` | `virsh pool-build <pool> [--overwrite]` |
| `pool-start` | `virsh pool-start <pool>` |
| `pool-destroy` | `virsh pool-destroy <pool>` |
| `pool-delete` | `virsh pool-delete <pool>` |
| `pool-undefine` | `virsh pool-undefine <pool>` |
| `pool-autostart` | `virsh pool-autostart <pool> [--disable]` |
| `pool-refresh` | `virsh pool-refresh <pool>` |
| `pool-edit` | `virsh pool-edit <pool>` |
| `find-storage-pool-sources-as` | `virsh find-storage-pool-sources-as iscsi [host]` |

### Storage Volume Commands

| Command | Usage |
|---|---|
| `vol-list` | `virsh vol-list <pool> [--details]` |
| `vol-info` | `virsh vol-info --pool <pool> <vol>` |
| `vol-dumpxml` | `virsh vol-dumpxml --pool <pool> <vol>` |
| `vol-path` | `virsh vol-path --pool <pool> <vol>` |
| `vol-create` | `virsh vol-create <pool> /path/vol.xml` |
| `vol-create-as` | `virsh vol-create-as <pool> <name> <capacity> [--allocation n] [--format f]` |
| `vol-clone` | `virsh vol-clone --pool <pool> <src-vol> <new-name>` |
| `vol-wipe` | `virsh vol-wipe --pool <pool> <vol> [--algorithm zero\|nnsa\|dod...]` |
| `vol-upload` | `virsh vol-upload --pool <pool> <vol> <local-file> [--offset n] [--length n]` |
| `vol-download` | `virsh vol-download --pool <pool> <vol> <local-file> [--offset n] [--length n]` |
| `vol-resize` | `virsh vol-resize --pool <pool> <vol> <capacity> [--delta] [--allocate] [--shrink]` |
| `vol-delete` | `virsh vol-delete <vol> --pool <pool>` |
| `vol-key` | `virsh vol-key --pool <pool> <vol>` |

### Block Device Commands (For Running VMs)

| Command | Usage |
|---|---|
| `domblklist` | `virsh domblklist <vm> [--details]` |
| `domblkinfo` | `virsh domblkinfo <vm> <target-dev>` |
| `domblkstat` | `virsh domblkstat <vm> [target-dev] [--human]` |
| `blockresize` | `virsh blockresize <vm> <disk-path> <size>` |
| `blockpull` | `virsh blockpull <vm> <disk-path> [--wait]` |
| `blockcommit` | `virsh blockcommit <vm> <disk-path> [--wait]` |
| `blkdeviotune` | `virsh blkdeviotune <vm> <target> --total-bytes-sec n [--live]` |
| `attach-disk` | `virsh attach-disk <vm> <src-path> <target> [--live] [--persistent]` |
| `detach-disk` | `virsh detach-disk <vm> <target> [--live] [--persistent]` |

---

## 23. Quick-Reference Cheat Sheet

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 POOL LIFECYCLE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 virsh pool-list --all                     List all pools
 virsh pool-info <pool>                    State, capacity, allocation
 virsh pool-define-as <n> <type> --target  Define persistent pool
 virsh pool-build <pool>                   Init storage (format/mkdir)
 virsh pool-start <pool>                   Start (mount/activate)
 virsh pool-autostart <pool>               Auto-start on libvirtd start
 virsh pool-destroy <pool>                 Stop (data intact)
 virsh pool-delete <pool>                  Delete storage (DATA GONE)
 virsh pool-undefine <pool>                Remove definition
 virsh pool-refresh <pool>                 Rescan for new volumes

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 POOL TYPE KEYWORDS (for pool-define-as)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 dir      Directory  (files: qcow2/raw)
 disk     Whole disk (GPT partitions as volumes)
 fs       Partition  (formatted, mounted to target dir)
 logical  LVM VG     (LVs as volumes)
 netfs    NFS share  (files on remote share)
 iscsi    iSCSI      (LUNs as volumes)
 gluster  GlusterFS  (files on Gluster volume)
 scsi     FC/vHBA    (SAN LUNs via NPIV)
 rbd      Ceph RBD   (Ubuntu: install libvirt-daemon-driver-storage-rbd)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 VOLUME MANAGEMENT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 virsh vol-list <pool> --details            List with sizes
 virsh vol-info --pool <pool> <vol>         Physical vs logical size
 virsh vol-create-as <pool> <name> <cap>    Create new volume
 virsh vol-clone --pool <pool> <src> <dst>  Clone a volume
 virsh vol-resize --pool <pool> <vol> <sz>  Resize volume
 virsh vol-wipe --pool <pool> <vol>         Secure wipe (zeroes)
 virsh vol-upload --pool <pool> <vol> <f>   Upload file → volume
 virsh vol-download --pool <pool> <vol> <f> Download volume → file
 virsh vol-delete <vol> --pool <pool>       Delete volume

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 qemu-img ESSENTIALS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 qemu-img create -f qcow2 disk.qcow2 20G     Create thin qcow2
 qemu-img create -f raw disk.raw 20G         Create raw image
 qemu-img create -f qcow2 -b base.qcow2 -F qcow2 overlay.qcow2 20G  COW
 qemu-img info disk.qcow2                    Inspect image
 qemu-img info --backing-chain overlay.qcow2 Show full chain
 qemu-img check disk.qcow2                   Check for errors
 qemu-img convert -f qcow2 -O raw in out     Convert format
 qemu-img convert -p -c -O qcow2 in out      Convert + compress
 qemu-img resize disk.qcow2 +10G             Grow image
 qemu-img commit overlay.qcow2               Merge overlay → base
 qemu-img snapshot -l disk.qcow2             List snapshots

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 LIVE DISK RESIZE (full workflow inside guest)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 HOST: virsh vol-resize --pool <p> <v> 30G
 HOST: virsh blockresize <vm> <disk-path> 30G
 GUEST: sudo growpart /dev/vda 1
 GUEST: sudo resize2fs /dev/vda1            (ext4)
 GUEST: sudo xfs_growfs /                   (xfs)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 DISK PERFORMANCE SETTINGS IN DOMAIN XML
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Best prod:  <driver name='qemu' type='qcow2' cache='none' io='native'/>
 Dev/test:   <driver name='qemu' type='qcow2' cache='writeback'/>
 LVM/block:  <driver name='qemu' type='raw'   cache='none' io='native'/>
 Bus:        <target dev='vda' bus='virtio'/>  (virtio = best)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 KEY UBUNTU vs RHEL DIFFERENCES (storage context)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Package manager  apt  (not yum/dnf)
 iSCSI initiator  open-iscsi  (not iscsi-initiator-utils)
 NFS server       nfs-kernel-server  (not nfs-utils)
 Security module  AppArmor  (not SELinux/virt_use_fusefs)
 AppArmor fix     /etc/apparmor.d/abstractions/libvirt-qemu
 Ceph/RBD         sudo apt install libvirt-daemon-driver-storage-rbd
 LVM packages     lvm2 (same package name)
 qemu-img pkg     qemu-utils  (not qemu-img directly)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

> **References:**
> - libvirt Storage Management: https://libvirt.org/storage.html
> - libvirt Storage Pool XML: https://libvirt.org/formatstorage.html
> - Ubuntu Server Guide — Storage: https://ubuntu.com/server/docs/storage-introduction
> - Ubuntu iSCSI: https://ubuntu.com/server/docs/service-iscsi
> - Ubuntu NFS: https://ubuntu.com/server/docs/service-nfs
> - qemu-img man page: `man qemu-img`
> - `man virsh` · `man lvcreate` · `man parted` · `man targetcli`
