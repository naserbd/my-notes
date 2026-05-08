# KVM on Ubuntu — Cloning Virtual Machines
### Engineer's Extensive Reference | Topic: VM Cloning Only
> Ubuntu 20.04 / 22.04 / 24.04 LTS · libvirt · virt-clone · virt-sysprep · virt-manager  
> Scope: Clone types → pre-clone preparation → virt-clone → templates → COW overlays → automation

---

## Table of Contents

1. [Clones vs Templates — Concepts](#1-clones-vs-templates--concepts)
2. [What Must Be Removed Before Cloning](#2-what-must-be-removed-before-cloning)
3. [Preflight Checks Before Any Clone Operation](#3-preflight-checks-before-any-clone-operation)
4. [Method A — Automated Preparation with virt-sysprep](#4-method-a--automated-preparation-with-virt-sysprep)
5. [Method B — Manual Preparation (Ubuntu-Specific Steps)](#5-method-b--manual-preparation-ubuntu-specific-steps)
6. [Cloning with virt-clone (CLI)](#6-cloning-with-virt-clone-cli)
7. [Cloning with virt-manager (GUI)](#7-cloning-with-virt-manager-gui)
8. [COW Overlay Cloning (Thin/Linked Clones)](#8-cow-overlay-cloning-thinlinked-clones)
9. [Creating and Using Templates](#9-creating-and-using-templates)
10. [Cloud Image-Based Cloning](#10-cloud-image-based-cloning)
11. [Post-Clone Configuration](#11-post-clone-configuration)
12. [Bulk Clone Automation Scripts](#12-bulk-clone-automation-scripts)
13. [Cloning VMs with Multiple Disks](#13-cloning-vms-with-multiple-disks)
14. [Troubleshooting Clone Issues](#14-troubleshooting-clone-issues)
15. [Full Option Reference Tables](#15-full-option-reference-tables)
16. [Quick-Reference Cheat Sheet](#16-quick-reference-cheat-sheet)

---

## 1. Clones vs Templates — Concepts

### 1.1 What Is a Clone?

A clone is an independent, fully functional copy of a virtual machine — it has its own disk image and its own libvirt XML definition. Once created, a clone is completely separate from its source; changes to the source do not affect the clone and vice versa.

```
Source VM ──── virt-clone ────► Clone VM (independent copy)
   │                                │
   │ disk.qcow2                     │ clone-disk.qcow2
   │ XML config                     │ New XML (new name, new MAC, new UUID)
```

**Clone use cases:**
- Deploy N identical web servers or worker nodes
- Create a test environment that mirrors production
- Distribute pre-configured VM images to other hosts or teams
- Quickly recover a known-good state (clone from a "clean" VM)

### 1.2 What Is a Template?

A template is conceptually just a VM that has been specifically prepared for repeated cloning. There is no special libvirt object type called "template" — it is a convention, not a technical distinction.

```
Template VM ──── virt-clone ────► Clone 1
(never booted,   │
never modified)  ├─── virt-clone ────► Clone 2
                 │
                 └─── virt-clone ────► Clone 3
```

**Template characteristics:**
- Fully configured OS + software
- Generalised (all unique IDs stripped)
- Kept shut off permanently — never booted after preparation
- Stored in a read-only or protected location

**Naming convention to keep sane:**

```bash
# Recommended naming structure
template-ubuntu22-base          # Bare OS template
template-ubuntu22-lamp          # OS + LAMP stack template
template-ubuntu22-k8s-node      # OS + Kubernetes node software

# Clones from templates
template-ubuntu22-lamp__clone-web01
template-ubuntu22-lamp__clone-web02
```

### 1.3 Key Differences Summary

| Property | Clone | Template |
|---|---|---|
| libvirt type | Regular VM | Regular VM (by convention) |
| Booted after creation? | Yes | No — kept shut off |
| Modified after creation? | Yes | No |
| Used to create further copies? | Rarely | Designed for it |
| Disk image | Independent copy | Source for cloning |
| Unique IDs present? | Yes (assigned during clone) | No (stripped during prep) |

---

## 2. What Must Be Removed Before Cloning

Cloning a VM without removing unique identifiers causes conflicts when both the source and clone run simultaneously on the same network. This is the most common cloning mistake.

### 2.1 The Three Levels of Uniqueness

```
┌─────────────────────────────────────────────────────────────┐
│ Level 1: Platform / Hypervisor Layer                        │
│  → MAC addresses (assigned by libvirt)                      │
│  → VM UUID (in libvirt XML)                                 │
│  → Virtual hardware serial numbers                          │
│  (virt-clone handles these automatically)                   │
├─────────────────────────────────────────────────────────────┤
│ Level 2: Guest OS Layer                                      │
│  → SSH host keys (/etc/ssh/ssh_host_*)                      │
│  → Machine ID (/etc/machine-id, /var/lib/dbus/machine-id)   │
│  → Hostname (/etc/hostname)                                  │
│  → Static IP configs (Netplan YAML files)                   │
│  → cloud-init state (/var/lib/cloud/)                       │
│  → udev persistent NIC rules                                │
│  → Ubuntu Pro / subscription registration                   │
│  → Snap device IDs                                          │
│  (must be removed manually OR via virt-sysprep)             │
├─────────────────────────────────────────────────────────────┤
│ Level 3: Application Layer                                   │
│  → License keys / activation codes                         │
│  → Application-specific UUIDs / node IDs                   │
│  → Database node identifiers                                │
│  → Cluster membership tokens                                │
│  (always manual — application-specific knowledge required)  │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Ubuntu-Specific Unique Items (Critical List)

| Item | Location | Problem if Not Removed | How to Remove |
|---|---|---|---|
| SSH host keys | `/etc/ssh/ssh_host_*` | SSH clients warn about host key conflict | `rm /etc/ssh/ssh_host_*` or virt-sysprep |
| Machine ID | `/etc/machine-id` | systemd, DHCP, dbus use this as system identity | Truncate to empty; systemd regenerates on boot |
| D-Bus machine ID | `/var/lib/dbus/machine-id` | Must match `/etc/machine-id` | Symlink or truncate |
| Hostname | `/etc/hostname` | Two VMs with the same hostname on the LAN | Edit or set via cloud-init |
| Netplan static IP | `/etc/netplan/*.yaml` | IP conflict on the network | Remove static IP / switch to DHCP |
| cloud-init state | `/var/lib/cloud/` | cloud-init skips first-boot config on clone | Remove entire directory |
| cloud-init instance data | `/var/lib/cloud/instances/` | Stale instance data from source VM | Remove |
| Snap unique IDs | `/var/lib/snapd/state.json` | Snap store authentication conflicts | virt-sysprep or manual reset |
| Ubuntu Pro registration | `pro status` | Clone appears as the same registered machine | `sudo pro detach` before cloning |
| DHCP leases | `/var/lib/dhcp/` | Stale lease state | Remove lease files |
| udev persistent rules | `/etc/udev/rules.d/70-persistent-net.rules` | If exists: NIC names may become eth1 instead of eth0 | Remove file |
| Journal machine ID bindings | `/var/log/journal/MACHINE-ID/` | Journal linked to old machine-id | Remove journal dir or let it auto-recreate |
| Audit logs | `/var/log/audit/` | Not a functional problem, but contains old host identity | Remove or truncate |
| User SSH known_hosts | `~/.ssh/known_hosts` | Not a conflict, but contains source VM's seen hosts | Optional remove |

---

## 3. Preflight Checks Before Any Clone Operation

```bash
# 1. Source VM must be SHUT OFF — never clone a running VM
virsh domstate <source-vm>
# Must return: shut off
# If running:
virsh shutdown <source-vm>
# Wait, then verify:
watch -n2 'virsh domstate <source-vm>'
# Force off if graceful shutdown hangs after 60s:
virsh destroy <source-vm>

# 2. Check disk image integrity before cloning
DISK=$(virsh domblklist <source-vm> --details | awk '/disk/{print $4}')
echo "Disk: $DISK"
sudo qemu-img check "$DISK"
# "No errors were found" is the required output

# 3. Check available disk space (clone needs as much as the source's virtual size)
sudo qemu-img info "$DISK" | grep -E 'virtual size|disk size'
df -h /var/lib/libvirt/images
# Ensure free space > virtual size of source disk

# 4. Confirm libvirt is running
virsh list --all | grep <source-vm>   # Must appear in the list

# 5. List all disks the source VM has (multi-disk VMs need all listed in virt-clone)
virsh domblklist <source-vm> --details
```

---

## 4. Method A — Automated Preparation with virt-sysprep

`virt-sysprep` is the recommended Ubuntu approach. It connects to an offline disk image and strips all unique identifiers in a single command. It is part of the `libguestfs-tools` package.

### 4.1 Install virt-sysprep

```bash
sudo apt install libguestfs-tools
virt-sysprep --version
```

### 4.2 What virt-sysprep Does by Default

Run `virt-sysprep --list-operations` to see every available operation:

```bash
virt-sysprep --list-operations
```

Key operations performed by default:

| Operation name | What it cleans |
|---|---|
| `ssh-hostkeys` | Removes `/etc/ssh/ssh_host_*` — regenerated on next boot |
| `machine-id` | Truncates `/etc/machine-id` — regenerated on next boot |
| `net-hwaddr` | Removes MAC address entries from network configs |
| `hostname` | Resets hostname to `localhost.localdomain` |
| `logfiles` | Clears log files (`/var/log/**`) |
| `bash-history` | Removes bash history for all users |
| `tmp-files` | Removes `/tmp/*` and `/var/tmp/*` |
| `utmp` | Clears login records |
| `udev-persistent-net` | Removes `/etc/udev/rules.d/70-persistent-net.rules` |
| `user-account` | (Optional, not default) Removes user accounts |
| `customize` | Applies any `--hostname`, `--password`, etc. flags |

### 4.3 Basic virt-sysprep Usage

```bash
# On an offline disk image directly
sudo virt-sysprep -a /var/lib/libvirt/images/source-vm.qcow2

# On a defined (but shut-off) VM by name
sudo virt-sysprep -d <source-vmname>

# Dry run — see what would happen without making changes
sudo virt-sysprep -a /var/lib/libvirt/images/source-vm.qcow2 \
  --dry-run

# Verbose output — see every file touched
sudo virt-sysprep -a /var/lib/libvirt/images/source-vm.qcow2 \
  --verbose
```

### 4.4 virt-sysprep with Customisation (Recommended)

Clean AND inject the new identity in a single pass:

```bash
sudo virt-sysprep -a /var/lib/libvirt/images/source-vm.qcow2 \
  --hostname template-ubuntu22 \
  --operations defaults \
  --firstboot-command 'dpkg-reconfigure openssh-server' \
  --firstboot-command 'cloud-init clean --logs --reboot'
```

### 4.5 Selective Operations — Run Only What You Need

```bash
# Run only specific operations (whitelist approach)
sudo virt-sysprep -a /var/lib/libvirt/images/source-vm.qcow2 \
  --operations ssh-hostkeys,machine-id,net-hwaddr,logfiles,bash-history

# Run all EXCEPT specific operations (blacklist approach)
sudo virt-sysprep -a /var/lib/libvirt/images/source-vm.qcow2 \
  --operations defaults \
  --no-network \
  --exclude-operations hostname,user-account
```

### 4.6 virt-sysprep for Ubuntu-Specific Cleanup

Ubuntu has items that the default operations may not cover. Add these explicitly:

```bash
sudo virt-sysprep -a /var/lib/libvirt/images/source-vm.qcow2 \
  --operations defaults \
  --firstboot-command 'rm -rf /var/lib/cloud/instances/*' \
  --firstboot-command 'cloud-init clean --logs' \
  --firstboot-command 'dpkg-reconfigure openssh-server' \
  --firstboot-command 'systemd-machine-id-setup' \
  --run-command 'truncate -s 0 /etc/machine-id' \
  --run-command 'rm -f /var/lib/dbus/machine-id' \
  --run-command 'ln -s /etc/machine-id /var/lib/dbus/machine-id' \
  --run-command 'rm -rf /var/lib/cloud/' \
  --run-command 'rm -f /etc/netplan/50-cloud-init.yaml' \
  --run-command 'rm -rf /var/lib/dhcp/*.leases' \
  --hostname template-ubuntu22
```

**What each extra command does:**

| Command | Why |
|---|---|
| `rm -rf /var/lib/cloud/instances/*` | Clears cloud-init instance data from source VM |
| `cloud-init clean --logs` | Resets cloud-init so it runs again on next boot |
| `dpkg-reconfigure openssh-server` | Regenerates SSH host keys (alternative to deleting them) |
| `systemd-machine-id-setup` | Generates a new machine-id on first boot |
| `truncate -s 0 /etc/machine-id` | Empties machine-id — systemd generates a new one on boot |
| `rm -f /var/lib/dbus/machine-id` | Removes the D-Bus copy — must match machine-id |
| `ln -s /etc/machine-id /var/lib/dbus/machine-id` | Symlink D-Bus id to system machine-id |
| `rm -rf /var/lib/cloud/` | Full cloud-init state removal |
| `rm -f /etc/netplan/50-cloud-init.yaml` | Remove cloud-init-generated Netplan config |
| `rm -rf /var/lib/dhcp/*.leases` | Clear stale DHCP lease state |

### 4.7 virt-sysprep --firstboot vs --run-command

| Flag | When it runs | Use for |
|---|---|---|
| `--run-command 'cmd'` | During virt-sysprep (on the host, while image is offline) | Removing files, truncating content |
| `--firstboot-command 'cmd'` | On next boot of the VM (inside the guest) | Anything needing a running OS: key regen, cloud-init reset |
| `--firstboot-install pkg` | On next boot (apt installs inside guest) | Adding packages to clones at first boot |

### 4.8 Inject SSH Authorised Keys During Sysprep

```bash
sudo virt-sysprep -a /var/lib/libvirt/images/source-vm.qcow2 \
  --ssh-inject ubuntu:file:/home/engineer/.ssh/id_ed25519.pub \
  --ssh-inject root:file:/home/engineer/.ssh/id_ed25519.pub \
  --operations defaults
```

### 4.9 Verify Sysprep Results

```bash
# Mount the image read-only and inspect key files
sudo guestmount -a /var/lib/libvirt/images/source-vm.qcow2 \
  -i --ro /mnt/inspect-vm

# Check the files that should be empty/absent
cat /mnt/inspect-vm/etc/machine-id          # Should be empty
ls /mnt/inspect-vm/etc/ssh/ssh_host_*       # Should return "No such file"
cat /mnt/inspect-vm/etc/hostname            # Should be generic
ls /mnt/inspect-vm/var/lib/cloud/           # Should be empty or absent

sudo guestunmount /mnt/inspect-vm
```

---

## 5. Method B — Manual Preparation (Ubuntu-Specific Steps)

Use this when you want fine-grained control or when virt-sysprep is unavailable. Boot the VM, log in, and run these commands inside the guest.

### Step 1: Remove SSH Host Keys

```bash
# Inside the guest — removes all SSH server identity keys
sudo rm -f /etc/ssh/ssh_host_*

# These are regenerated automatically by openssh-server on next boot
# To trigger regeneration manually on the same boot:
sudo dpkg-reconfigure openssh-server
# OR
sudo ssh-keygen -A        # Generates all missing key types
```

> **Why this matters:** Every SSH client that connected to the source VM stores its host key. If the clone reuses the same key, SSH clients will refuse the connection with a "host key verification failed" warning.

### Step 2: Reset the Machine ID

The machine-id is used by systemd for service identifiers, by DHCP clients for client identity, by D-Bus, and by journald for log partitioning. All clones sharing the same machine-id receive the same DHCP lease IP.

```bash
# Inside the guest
sudo truncate -s 0 /etc/machine-id

# Remove the D-Bus copy (symlink it to the system one instead)
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id

# On next boot, systemd-machine-id-setup generates a new unique ID
# To trigger immediately (optional, generates right now):
sudo systemd-machine-id-setup
cat /etc/machine-id     # Should show a new 32-char hex string
```

### Step 3: Remove cloud-init State

If the source VM used cloud-init, its state must be cleared so cloud-init re-runs on the clone's first boot.

```bash
# Inside the guest
sudo cloud-init clean --logs --reboot
# This clears state AND reboots immediately — use carefully

# Or clear state without rebooting:
sudo cloud-init clean
sudo rm -rf /var/lib/cloud/instances/
sudo rm -rf /var/lib/cloud/sem/

# Verify cloud-init will re-run on next boot
sudo cloud-init status
# Should show: "disabled" or "not run" after cleaning
```

### Step 4: Remove / Reset the Hostname

```bash
# Inside the guest
sudo hostnamectl set-hostname template-ubuntu22

# Also edit /etc/hosts to match
sudo nano /etc/hosts
# Change: 127.0.1.1   old-hostname
# To:     127.0.1.1   template-ubuntu22
```

### Step 5: Remove Network Configuration Unique Details (Ubuntu/Netplan)

Ubuntu uses **Netplan** YAML files — not `ifcfg-*` scripts. Strip unique network details:

```bash
# Inside the guest — list Netplan configs
ls /etc/netplan/

# Example: remove cloud-init generated network config (most common on Ubuntu)
sudo rm -f /etc/netplan/50-cloud-init.yaml

# If you have a custom Netplan config with static IPs, edit it:
sudo nano /etc/netplan/01-netcfg.yaml
```

**Before (static IP — must be removed):**

```yaml
# /etc/netplan/01-netcfg.yaml  — BEFORE (has unique static config)
network:
  version: 2
  ethernets:
    ens3:
      addresses: [192.168.1.50/24]   # ← REMOVE (unique)
      gateway4: 192.168.1.1          # ← REMOVE (unique)
      nameservers:
        addresses: [8.8.8.8]
      dhcp4: false
```

**After (DHCP — safe to clone):**

```yaml
# /etc/netplan/01-netcfg.yaml  — AFTER (generic DHCP config)
network:
  version: 2
  ethernets:
    ens3:
      dhcp4: true
```

```bash
# Apply and verify
sudo netplan apply
ip addr show ens3
```

### Step 6: Remove DHCP Lease State

```bash
# Inside the guest
sudo rm -f /var/lib/dhcp/*.leases
sudo rm -f /var/lib/dhcpcd5/*.lease    # If dhcpcd is used
```

### Step 7: Remove Ubuntu Pro Registration

If the source VM was registered with Ubuntu Pro (formerly Ubuntu Advantage):

```bash
# Check if registered
sudo pro status

# If registered — detach before cloning
# Option A: Source VM will be decommissioned (full detach)
sudo pro detach

# Option B: Source VM will keep running (clean only, no server-side detach)
sudo pro disable all
sudo rm -rf /var/lib/ubuntu-advantage/
sudo rm -f /etc/ubuntu-advantage/uaclient.conf
```

> After cloning, each clone should be independently registered with Ubuntu Pro using `sudo pro attach <token>`.

### Step 8: Remove Snap Device-Specific State (Optional)

```bash
# Inside the guest — Snap instances have a per-device ID
sudo snap remove --purge snapd      # Nuclear option — removes snapd entirely
# Or more surgically:
sudo systemctl stop snapd
sudo rm -f /var/lib/snapd/state.json
sudo systemctl start snapd
# Snap will re-initialise with a new device ID on next start
```

### Step 9: Remove User-Specific State

```bash
# Inside the guest
# Clear bash history for all users
sudo truncate -s 0 /root/.bash_history
sudo truncate -s 0 /home/ubuntu/.bash_history

# Clear SSH known_hosts (optional — not a conflict, just stale)
sudo rm -f /root/.ssh/known_hosts
sudo rm -f /home/ubuntu/.ssh/known_hosts

# Clear logs
sudo find /var/log -type f -exec truncate -s 0 {} \;

# Clear temp files
sudo rm -rf /tmp/* /var/tmp/*
```

### Step 10: Remove udev Persistent NIC Rules

On older Ubuntu versions udev may have created a persistent NIC name rule:

```bash
# Inside the guest
sudo rm -f /etc/udev/rules.d/70-persistent-net.rules

# On Ubuntu 20.04+ this file rarely exists, but check anyway
ls /etc/udev/rules.d/ | grep net
```

### Step 11: Configure First-Boot Behaviour

On Ubuntu, cloud-init handles first-boot configuration. Ensure it will run:

```bash
# Inside the guest — verify cloud-init is installed
cloud-init --version

# Set it to run on next boot
sudo cloud-init clean

# Optionally create a minimal user-data for the clone
# (This is set up in your cloud-init infrastructure, not on the VM directly)
```

### Step 12: Final Cleanup and Shutdown

```bash
# Inside the guest — final sweep
sudo apt autoremove -y
sudo apt clean
sudo fstrim -av                  # TRIM unused blocks (qcow2 image shrinks)

# Clear shell history one more time
history -c && history -w

# Shut down cleanly
sudo shutdown -h now
```

---

## 6. Cloning with virt-clone (CLI)

`virt-clone` creates a new VM definition and copies disk images. It automatically assigns a new UUID, new MAC address(es), and a new name.

### 6.1 Install virt-clone

```bash
# virt-clone is part of the virtinst package
sudo apt install virtinst
virt-clone --version
```

### 6.2 Source VM Must Be Shut Off

```bash
virsh domstate <source-vm>
# Must show: shut off

# If running:
virsh shutdown <source-vm>

# Verify (wait for it):
watch -n2 'virsh domstate <source-vm>'
```

### 6.3 Simplest Clone — Auto-Generated Name and Disk Path

```bash
virt-clone --original <source-vm> --auto-clone

# What --auto-clone does:
# → Appends "-clone" to the VM name → source-vm-clone
# → Appends "-clone.qcow2" to the disk path
# → Generates new UUID and MAC addresses automatically
```

### 6.4 Named Clone with Explicit Disk Path

```bash
virt-clone \
  --original demo \
  --name newdemo \
  --file /var/lib/libvirt/images/newdemo.qcow2
```

### 6.5 Full Command with All Common Options

```bash
virt-clone \
  --connect qemu:///system \
  --original source-vm \
  --name prod-web01 \
  --file /var/lib/libvirt/images/prod-web01.qcow2 \
  --check path_in_use=off

# --connect qemu:///system  → Use system-level libvirt (default; use sudo or be in libvirt group)
# --original source-vm      → Name of the source VM (must be shut off)
# --name prod-web01         → Name of the new clone
# --file /path/to/new.qcow2 → Where to write the cloned disk image
# --check path_in_use=off   → Skip warning if the target path already exists
```

### 6.6 Clone with a Different Disk Format

```bash
# Clone and convert from qcow2 to raw (or any other combination)
virt-clone \
  --original source-vm \
  --name prod-db01 \
  --file /var/lib/libvirt/images/prod-db01.raw
# virt-clone detects format from the source; specify explicitly if needed
```

### 6.7 Rename the MAC Address of the Clone

By default, virt-clone generates a random MAC. You can specify one:

```bash
virt-clone \
  --original source-vm \
  --name prod-web01 \
  --file /var/lib/libvirt/images/prod-web01.qcow2 \
  --mac 52:54:00:AA:BB:CC

# Let libvirt auto-assign (default — recommended)
--mac RANDOM
```

### 6.8 Clone Without Copying Disk (Shallow Clone — Rare)

For debugging the XML definition only, without duplicating disk data:

```bash
virt-clone \
  --original source-vm \
  --name debug-vm \
  --preserve-data \
  --file /var/lib/libvirt/images/source-vm.qcow2
# WARNING: Both VMs will share the same disk — NEVER boot both simultaneously
# Use only for XML inspection, then undefine the clone
```

### 6.9 Check Progress of a Large Clone

Cloning copies the entire virtual disk. For large disks, monitor progress:

```bash
# In another terminal, watch the target file grow
watch -n2 'ls -lh /var/lib/libvirt/images/prod-web01.qcow2'

# Or use pv for byte-level progress (if cloning manually with cp)
sudo apt install pv
pv /var/lib/libvirt/images/source.qcow2 > /var/lib/libvirt/images/clone.qcow2

# Or check I/O in real-time
iostat -xh 2
```

### 6.10 What virt-clone Creates

After a successful clone:

```bash
# 1. A new disk image
ls -lh /var/lib/libvirt/images/prod-web01.qcow2

# 2. A new VM definition (XML)
virsh dumpxml prod-web01
# Shows: new <uuid>, new <mac address='...'>, new <name>

# 3. The VM appears in virsh list
virsh list --all | grep prod-web01

# 4. The source VM is unchanged
virsh domstate source-vm    # still: shut off
```

---

## 7. Cloning with virt-manager (GUI)

### 7.1 Steps to Clone via virt-manager

1. Open virt-manager: `virt-manager &`
2. Right-click the source VM → select **"Clone"**
3. In the Clone dialog:
   - **Name:** Enter the new VM name
   - **Storage:** For each disk, choose:
     - "Clone this disk" (full copy — independent)
     - "Share disk with original" (dangerous — do not use for production)
     - "New disk path" — browse to set explicitly
   - **MAC:** Click "Details" to see/override the generated MAC
4. Click **"Clone"** — progress bar appears
5. New VM appears in the VM list when complete

### 7.2 virt-manager vs virt-clone — When to Use Which

| Scenario | Use |
|---|---|
| Single ad-hoc clone | Either — virt-manager is easier |
| Bulk cloning (N VMs) | virt-clone in a script |
| Automated provisioning pipeline | virt-clone in CI/CD or Ansible |
| Remote headless host | virt-clone (SSH in) |
| Quick one-off on a desktop host | virt-manager |

---

## 8. COW Overlay Cloning (Thin/Linked Clones)

Copy-on-Write overlays are an alternative to full disk copies. An overlay stores only the **difference** from a base image. This is dramatically faster and uses far less disk space — but all overlays depend on the base image remaining intact and unmodified.

### 8.1 How COW Overlays Work

```
base-ubuntu22.qcow2  (read-only, never modified after creation)
         │
         ├──── overlay-web01.qcow2  (stores only web01's changes)
         ├──── overlay-web02.qcow2  (stores only web02's changes)
         └──── overlay-db01.qcow2   (stores only db01's changes)

Disk space used:
  base image:  600 MB
  each overlay: 50–200 MB (only changed blocks)
  vs full clone: 20 GB per VM
```

### 8.2 Create a COW Overlay

```bash
# Step 1: Prepare and lock down the base image
BASE="/var/lib/libvirt/images/base/template-ubuntu22.qcow2"
sudo chmod 444 "$BASE"           # Make read-only to prevent accidental modification

# Step 2: Create an overlay
sudo qemu-img create \
  -f qcow2 \
  -b "$BASE" \
  -F qcow2 \
  /var/lib/libvirt/images/overlay-web01.qcow2 \
  20G

# Step 3: Verify the backing file chain
qemu-img info --backing-chain /var/lib/libvirt/images/overlay-web01.qcow2
# Shows: overlay → base → (end)
```

### 8.3 Create a VM Using the Overlay

```bash
virt-install \
  --name web01 \
  --memory 2048 \
  --vcpus 2 \
  --disk /var/lib/libvirt/images/overlay-web01.qcow2,bus=virtio \
  --import \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --noautoconsole
```

### 8.4 COW Clone — Full Workflow

```bash
#!/bin/bash
# Full COW clone workflow

BASE="/var/lib/libvirt/images/base/template-ubuntu22.qcow2"
OVERLAY_DIR="/var/lib/libvirt/images"
VM_NAME="web01"
OVERLAY="${OVERLAY_DIR}/overlay-${VM_NAME}.qcow2"

# Create overlay
sudo qemu-img create -f qcow2 -b "$BASE" -F qcow2 "$OVERLAY" 20G

# Import as VM
sudo virt-install \
  --name "$VM_NAME" \
  --memory 2048 \
  --vcpus 2 \
  --disk "${OVERLAY},bus=virtio" \
  --import \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --noautoconsole \
  --autostart
```

### 8.5 Collapse an Overlay into an Independent Image (Flatten)

When you want the clone to become fully independent (no longer depends on the base):

```bash
# Convert overlay to a standalone qcow2 (copies base + changes into one file)
sudo qemu-img convert \
  -f qcow2 \
  -O qcow2 \
  /var/lib/libvirt/images/overlay-web01.qcow2 \
  /var/lib/libvirt/images/standalone-web01.qcow2

# Check — should show NO backing file
qemu-img info /var/lib/libvirt/images/standalone-web01.qcow2 | grep backing
# (no output = no backing file = fully independent)

# Update the VM to use the new standalone image
virsh edit web01
# Change <source file='...overlay-web01.qcow2'/> to <source file='...standalone-web01.qcow2'/>
```

### 8.6 COW Advantages vs Disadvantages

| Aspect | COW Overlay | Full Clone (virt-clone) |
|---|---|---|
| Creation time | Seconds | Minutes (disk copy) |
| Disk space | ~50–300 MB overhead per clone | Full disk size per clone |
| Independence | Depends on base image | Fully independent |
| Base image safety | Base must never be modified | Source can be used freely |
| Performance | Slight read overhead for unchanged blocks | Full native performance |
| Backup complexity | Must back up base + all overlays | Back up each disk independently |
| Best for | Dev/test, CI/CD, ephemeral VMs | Production, long-lived VMs |

---

## 9. Creating and Using Templates

### 9.1 Building a Template from Scratch

The recommended workflow:

```
1. Create a fresh VM and install Ubuntu
2. Configure the OS and install software
3. Generalise (virt-sysprep)
4. Protect the template disk
5. Clone from the template whenever needed
```

**Step 1–2: Build and configure the VM (standard virt-install)**

```bash
# Create the source VM
virt-install \
  --name template-builder \
  --memory 4096 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/template-ubuntu22-lamp.qcow2,\
size=20,format=qcow2,bus=virtio \
  --cdrom /var/lib/libvirt/boot/ubuntu-22.04-live-server-amd64.iso \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --graphics vnc,listen=0.0.0.0 \
  --noautoconsole

# ... install Ubuntu, install LAMP stack, configure ...

# Inside the VM: install your software
sudo apt install -y apache2 mysql-server php libapache2-mod-php
sudo systemctl enable apache2 mysql
```

**Step 3: Generalise with virt-sysprep**

```bash
# Shut down the builder VM
virsh shutdown template-builder

# Run virt-sysprep to strip all unique identifiers
sudo virt-sysprep -d template-builder \
  --operations defaults \
  --run-command 'truncate -s 0 /etc/machine-id' \
  --run-command 'rm -f /var/lib/dbus/machine-id' \
  --run-command 'ln -s /etc/machine-id /var/lib/dbus/machine-id' \
  --run-command 'cloud-init clean' \
  --run-command 'rm -rf /var/lib/cloud/' \
  --run-command 'apt-get clean -y' \
  --hostname template-ubuntu22-lamp
```

**Step 4: Rename and protect the template**

```bash
# Optional: rename the VM definition to reflect template status
virsh dumpxml template-builder > /tmp/template-builder.xml
sed -i 's/<name>template-builder</<name>template-ubuntu22-lamp</' /tmp/template-builder.xml
virsh undefine template-builder
virsh define /tmp/template-builder.xml

# Protect the disk image from accidental modification
sudo chmod 444 /var/lib/libvirt/images/template-ubuntu22-lamp.qcow2

# Verify the template VM is shut off and stays that way
virsh domstate template-ubuntu22-lamp
# Output: shut off
```

### 9.2 Cloning from a Template

```bash
# Full clone (independent copy)
virt-clone \
  --original template-ubuntu22-lamp \
  --name web-prod-01 \
  --file /var/lib/libvirt/images/web-prod-01.qcow2

# COW overlay (thin, fast)
sudo qemu-img create -f qcow2 \
  -b /var/lib/libvirt/images/template-ubuntu22-lamp.qcow2 \
  -F qcow2 \
  /var/lib/libvirt/images/overlay-web-prod-01.qcow2 \
  20G
```

### 9.3 Template Inventory — Recommended Structure

```bash
/var/lib/libvirt/images/
├── base/                          # Protected base/template images
│   ├── template-ubuntu22-base.qcow2       # Bare OS only
│   ├── template-ubuntu22-lamp.qcow2       # OS + LAMP
│   ├── template-ubuntu22-k8s-node.qcow2   # OS + k8s components
│   └── template-ubuntu24-base.qcow2
└── vms/                           # Running VM images
    ├── web-prod-01.qcow2
    ├── web-prod-02.qcow2
    ├── db-prod-01.qcow2
    └── overlay-dev-01.qcow2       # COW overlays for dev

# Keep a manifest
cat > /var/lib/libvirt/images/base/MANIFEST.md <<'EOF'
| Image | OS | Software | Created | Last sysprep |
|---|---|---|---|---|
| template-ubuntu22-base.qcow2 | Ubuntu 22.04 | Base only | 2024-01-10 | 2024-01-10 |
| template-ubuntu22-lamp.qcow2 | Ubuntu 22.04 | Apache/MySQL/PHP 8.1 | 2024-01-12 | 2024-01-12 |
EOF
```

### 9.4 Refresh / Update a Template

When OS updates are needed on the template:

```bash
# Step 1: Make a working copy to update (never modify the template directly)
sudo cp /var/lib/libvirt/images/base/template-ubuntu22-lamp.qcow2 \
        /var/lib/libvirt/images/template-ubuntu22-lamp-update.qcow2
sudo chmod 644 /var/lib/libvirt/images/template-ubuntu22-lamp-update.qcow2

# Step 2: Import as a temporary VM
virt-install \
  --name template-update-vm \
  --memory 2048 --vcpus 2 \
  --disk /var/lib/libvirt/images/template-ubuntu22-lamp-update.qcow2,bus=virtio \
  --import --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --noautoconsole

# Step 3: SSH in and update
virsh net-dhcp-leases default    # Get IP
ssh ubuntu@<IP>
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y && sudo apt clean
sudo shutdown -h now

# Step 4: Re-run virt-sysprep on the updated image
sudo virt-sysprep -d template-update-vm --operations defaults \
  --run-command 'truncate -s 0 /etc/machine-id' \
  --run-command 'rm -f /var/lib/dbus/machine-id' \
  --run-command 'ln -s /etc/machine-id /var/lib/dbus/machine-id' \
  --run-command 'cloud-init clean' \
  --run-command 'rm -rf /var/lib/cloud/'

# Step 5: Replace the old template
virsh undefine template-update-vm
sudo mv /var/lib/libvirt/images/base/template-ubuntu22-lamp.qcow2 \
        /var/lib/libvirt/images/base/template-ubuntu22-lamp.qcow2.bak
sudo mv /var/lib/libvirt/images/template-ubuntu22-lamp-update.qcow2 \
        /var/lib/libvirt/images/base/template-ubuntu22-lamp.qcow2
sudo chmod 444 /var/lib/libvirt/images/base/template-ubuntu22-lamp.qcow2
```

---

## 10. Cloud Image-Based Cloning

Ubuntu cloud images + cloud-init is the most production-grade approach to VM cloning. Each "clone" is deployed from the same base image with a per-VM cloud-init `user-data` that handles all identity configuration at first boot.

### 10.1 Why Cloud Images Are Ideal for Cloning

| Aspect | Traditional virt-clone | Cloud image + cloud-init |
|---|---|---|
| VM identity setup | Manual or virt-sysprep | Declarative YAML (user-data) |
| Hostname per clone | Must SSH in and set | Set in user-data |
| SSH keys per clone | Must inject manually | Declared in user-data |
| Speed | Slow (full disk copy) | Fast (COW + seconds to boot) |
| Repeatability | Manual steps | Fully declarative |
| Scale | 1–10 VMs comfortably | 100+ VMs easily |

### 10.2 Base Image Preparation (Once Only)

```bash
# Download Ubuntu cloud image
wget -P /var/lib/libvirt/images/base/ \
  https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Lock it down
sudo chmod 444 /var/lib/libvirt/images/base/jammy-server-cloudimg-amd64.img
```

### 10.3 Per-VM Deployment Pattern

```bash
VM_NAME="web-prod-01"
BASE="/var/lib/libvirt/images/base/jammy-server-cloudimg-amd64.img"

# 1. Create a COW overlay for this VM
sudo qemu-img create -f qcow2 -b "$BASE" -F qcow2 \
  /var/lib/libvirt/images/${VM_NAME}.qcow2 20G

# 2. Create per-VM cloud-init user-data
cat > /tmp/user-data-${VM_NAME} <<EOF
#cloud-config
hostname: ${VM_NAME}
manage_etc_hosts: true
users:
  - name: ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - "$(cat ~/.ssh/id_ed25519.pub)"
packages: [qemu-guest-agent, curl]
runcmd:
  - systemctl enable --now qemu-guest-agent
EOF

cat > /tmp/meta-data-${VM_NAME} <<EOF
instance-id: ${VM_NAME}-$(date +%s)
local-hostname: ${VM_NAME}
EOF

cloud-localds /tmp/seed-${VM_NAME}.iso \
  /tmp/user-data-${VM_NAME} \
  /tmp/meta-data-${VM_NAME}

# 3. Boot the VM
sudo virt-install \
  --name "$VM_NAME" \
  --memory 2048 --vcpus 2 \
  --disk /var/lib/libvirt/images/${VM_NAME}.qcow2,bus=virtio \
  --disk /tmp/seed-${VM_NAME}.iso,device=cdrom \
  --import \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --graphics none \
  --noautoconsole \
  --autostart
```

---

## 11. Post-Clone Configuration

After cloning, the clone needs its own identity. Do this on first boot.

### 11.1 Set Hostname

```bash
# Inside the clone
sudo hostnamectl set-hostname new-clone-hostname

# Update /etc/hosts
sudo sed -i "s/127.0.1.1.*/127.0.1.1\tnew-clone-hostname/" /etc/hosts

# Verify
hostname
hostnamectl status
```

### 11.2 Verify SSH Host Keys Were Regenerated

```bash
# Inside the clone — keys should exist now (regenerated on first boot)
ls -la /etc/ssh/ssh_host_*
# Should show: ssh_host_ed25519_key, ssh_host_rsa_key, etc.

# Verify fingerprints are different from the source (compare hashes)
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

### 11.3 Verify Machine ID is Unique

```bash
# On source VM
virsh console source-vm
cat /etc/machine-id

# On clone (in another terminal)
virsh console clone-vm
cat /etc/machine-id

# The two hex strings must be DIFFERENT
```

### 11.4 Configure Netplan Static IP (If Needed)

```bash
# Inside the clone
sudo nano /etc/netplan/01-netcfg.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens3:
      addresses: [192.168.1.101/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
      dhcp4: false
```

```bash
sudo netplan apply
ip addr show ens3
```

### 11.5 Register with Ubuntu Pro (If Applicable)

```bash
# Inside each clone that needs Ubuntu Pro
sudo pro attach <YOUR_TOKEN>
sudo pro status
```

### 11.6 Run cloud-init if Not Already Done

```bash
# Inside the clone — verify cloud-init ran
cloud-init status
# Output: status: done  ← means cloud-init completed successfully

# If status is "disabled", trigger a run
sudo cloud-init init
sudo cloud-init modules --mode=config
sudo cloud-init modules --mode=final
```

### 11.7 Post-Clone Verification Checklist

```bash
# Run this inside each clone

echo "=== Hostname ===" && hostname
echo "=== Machine ID ===" && cat /etc/machine-id
echo "=== IP Address ===" && ip -4 addr show ens3
echo "=== SSH Host Keys ===" && ls -la /etc/ssh/ssh_host_* | awk '{print $NF, $5}'
echo "=== cloud-init Status ===" && cloud-init status
echo "=== OS Release ===" && lsb_release -a 2>/dev/null
echo "=== Disk Space ===" && df -h /
```

---

## 12. Bulk Clone Automation Scripts

### 12.1 Full Clone — N VMs from Template

```bash
#!/bin/bash
# Bulk full clone: creates N independent VM copies from a template

TEMPLATE="template-ubuntu22-lamp"
TEMPLATE_DISK="/var/lib/libvirt/images/base/template-ubuntu22-lamp.qcow2"
IMG_DIR="/var/lib/libvirt/images/vms"
VM_PREFIX="web-prod"
COUNT=5
MEMORY=4096
VCPUS=2
DISK_SIZE=30G

mkdir -p "$IMG_DIR"

for i in $(seq -f "%02g" 1 $COUNT); do
  VM_NAME="${VM_PREFIX}-${i}"
  VM_DISK="${IMG_DIR}/${VM_NAME}.qcow2"

  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "→ Cloning: ${VM_NAME}"

  # Verify template is shut off
  STATE=$(virsh domstate "$TEMPLATE" 2>/dev/null)
  if [[ "$STATE" != "shut off" ]]; then
    echo "ERROR: Template '$TEMPLATE' is not shut off. Aborting."
    exit 1
  fi

  # Clone
  virt-clone \
    --connect qemu:///system \
    --original "$TEMPLATE" \
    --name "$VM_NAME" \
    --file "$VM_DISK"

  if [[ $? -ne 0 ]]; then
    echo "ERROR: virt-clone failed for $VM_NAME"
    continue
  fi

  # Adjust resources if different from template
  virsh setmaxmem "$VM_NAME" $((MEMORY * 1024)) --config
  virsh setmem "$VM_NAME" $((MEMORY * 1024)) --config
  virsh setvcpus "$VM_NAME" $VCPUS --config --maximum
  virsh setvcpus "$VM_NAME" $VCPUS --config

  # Set autostart
  virsh autostart "$VM_NAME"

  echo "✓ ${VM_NAME} cloned successfully"
  echo "  Disk: ${VM_DISK}"
  echo "  MAC: $(virsh domiflist $VM_NAME | awk 'NR>2{print $5}')"
done

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Starting all clones..."
for i in $(seq -f "%02g" 1 $COUNT); do
  virsh start "${VM_PREFIX}-${i}" && echo "✓ ${VM_PREFIX}-${i} started"
  sleep 3    # Stagger starts to avoid DHCP collision
done

echo ""
echo "Waiting 30s for DHCP leases..."
sleep 30
echo "=== DHCP Leases ==="
virsh net-dhcp-leases default
```

### 12.2 COW Overlay — N VMs from Cloud Base Image

```bash
#!/bin/bash
# Fast thin-clone: COW overlays + cloud-init per VM

BASE="/var/lib/libvirt/images/base/jammy-server-cloudimg-amd64.img"
IMG_DIR="/var/lib/libvirt/images/vms"
SEED_DIR="/tmp/cloud-seeds"
SSH_PUB="$HOME/.ssh/id_ed25519.pub"
VM_PREFIX="node"
COUNT=5
MEMORY=2048
VCPUS=2
DISK_SIZE=20G

mkdir -p "$IMG_DIR" "$SEED_DIR"

# Validate base image
if [[ ! -f "$BASE" ]]; then
  echo "ERROR: Base image not found: $BASE"
  exit 1
fi

if [[ ! -f "$SSH_PUB" ]]; then
  echo "ERROR: SSH public key not found: $SSH_PUB"
  exit 1
fi

SSH_KEY=$(cat "$SSH_PUB")

for i in $(seq -f "%02g" 1 $COUNT); do
  VM_NAME="${VM_PREFIX}-${i}"
  VM_DISK="${IMG_DIR}/${VM_NAME}.qcow2"
  SEED="${SEED_DIR}/seed-${VM_NAME}.iso"
  USERDATA="${SEED_DIR}/user-data-${VM_NAME}"
  METADATA="${SEED_DIR}/meta-data-${VM_NAME}"

  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "→ Deploying: ${VM_NAME}"

  # Create COW overlay
  sudo qemu-img create -f qcow2 -b "$BASE" -F qcow2 "$VM_DISK" "$DISK_SIZE"
  [[ $? -ne 0 ]] && echo "ERROR: qemu-img failed for $VM_NAME" && continue

  # Write cloud-init user-data
  cat > "$USERDATA" <<EOF
#cloud-config
hostname: ${VM_NAME}
manage_etc_hosts: true
fqdn: ${VM_NAME}.internal
users:
  - name: ubuntu
    groups: [sudo, adm]
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - "${SSH_KEY}"
ssh_pwauth: false
packages:
  - qemu-guest-agent
  - curl
runcmd:
  - systemctl enable --now qemu-guest-agent
  - echo "Provisioned: \$(date)" > /etc/provision-stamp
EOF

  # Write meta-data
  cat > "$METADATA" <<EOF
instance-id: ${VM_NAME}-$(date +%s)
local-hostname: ${VM_NAME}
EOF

  # Build seed ISO
  cloud-localds "$SEED" "$USERDATA" "$METADATA"

  # Create VM
  sudo virt-install \
    --name "$VM_NAME" \
    --memory $MEMORY \
    --vcpus $VCPUS \
    --disk "${VM_DISK},bus=virtio,cache=none" \
    --disk "${SEED},device=cdrom" \
    --import \
    --os-variant ubuntu22.04 \
    --network network=default,model=virtio \
    --graphics none \
    --noautoconsole \
    --autostart

  echo "✓ ${VM_NAME} deployed"
  sleep 2
done

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Waiting 45s for cloud-init to complete..."
sleep 45

echo ""
echo "=== VM Status ==="
virsh list --all | grep "$VM_PREFIX"

echo ""
echo "=== DHCP Leases ==="
virsh net-dhcp-leases default

echo ""
echo "=== SSH Test ==="
for i in $(seq -f "%02g" 1 $COUNT); do
  VM_NAME="${VM_PREFIX}-${i}"
  MAC=$(virsh domiflist "$VM_NAME" | awk 'NR>2{print $5}')
  IP=$(virsh net-dhcp-leases default | grep "$MAC" | awk '{print $5}' | cut -d/ -f1)
  if [[ -n "$IP" ]]; then
    RESULT=$(ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 \
      ubuntu@$IP 'hostname && cloud-init status' 2>/dev/null)
    echo "  ${VM_NAME} (${IP}): $RESULT"
  else
    echo "  ${VM_NAME}: No IP yet"
  fi
done
```

---

## 13. Cloning VMs with Multiple Disks

VMs with more than one disk need all disks specified in the virt-clone command.

### 13.1 Identify All Disks of the Source VM

```bash
virsh domblklist <source-vm> --details
# Type   Device   Target   Source
# file   disk     vda      /var/lib/libvirt/images/source-vm-os.qcow2
# file   disk     vdb      /var/lib/libvirt/images/source-vm-data.qcow2
# file   cdrom    sda      -
```

### 13.2 Clone a Two-Disk VM

```bash
# One --file per disk (order matters — matches disk order in domblklist)
virt-clone \
  --original source-vm \
  --name clone-vm \
  --file /var/lib/libvirt/images/clone-vm-os.qcow2 \
  --file /var/lib/libvirt/images/clone-vm-data.qcow2
```

### 13.3 Clone a VM with Many Disks

```bash
# For VMs with 4+ disks
virt-clone \
  --original db-source \
  --name db-clone-01 \
  --file /var/lib/libvirt/images/db-clone-01-os.qcow2 \
  --file /var/lib/libvirt/images/db-clone-01-data.qcow2 \
  --file /var/lib/libvirt/images/db-clone-01-log.qcow2 \
  --file /var/lib/libvirt/images/db-clone-01-backup.qcow2
```

### 13.4 Skip Cloning a Read-Only / Shared Disk

If one disk is shared data (ISO, read-only reference data), skip it:

```bash
virt-clone \
  --original source-vm \
  --name clone-vm \
  --file /var/lib/libvirt/images/clone-vm-os.qcow2 \
  --skip-copy vdb    # Skip the second disk (vdb) — the clone references it in-place
```

> ⚠️ `--skip-copy` means both source and clone share the same disk image for `vdb`. Only safe for genuinely read-only data disks.

---

## 14. Troubleshooting Clone Issues

### "Domain with duplicate name already exists"

```bash
# Check existing VMs
virsh list --all

# If a previous failed clone left a stale definition:
virsh undefine stale-clone-name

# Then retry virt-clone
```

### "error: Disk /path/to/clone.qcow2 already exists"

```bash
# Remove the file from the failed attempt
sudo rm -f /var/lib/libvirt/images/failed-clone.qcow2

# Or tell virt-clone to overwrite
virt-clone ... --check path_in_use=off
```

### "error: Source VM must not be running"

```bash
virsh shutdown <source-vm>
# Wait...
virsh domstate <source-vm>
# Must show: shut off
# Then retry virt-clone
```

### Clone Boots but Gets the Same IP as Source

```bash
# Cause: machine-id was not cleared before cloning
# DHCP servers use machine-id as client identifier → same ID = same lease

# Fix (inside the clone):
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
sudo reboot
# After reboot: new machine-id is generated → different DHCP lease
```

### Clone Has No Network / NIC Not Found

```bash
# Cause 1: Netplan config references old MAC or interface name
# Inside the clone:
ip link show                     # See actual NIC name (ens3? enp1s0? eth0?)
ls /etc/netplan/
cat /etc/netplan/*.yaml          # Check for MAC address references

# Remove MAC constraint from Netplan
sudo nano /etc/netplan/01-netcfg.yaml
# Remove any "match:" section with macaddress
sudo netplan apply

# Cause 2: Old udev rule mapping old MAC to eth0/eth1
ls /etc/udev/rules.d/ | grep net
sudo rm -f /etc/udev/rules.d/70-persistent-net.rules
sudo reboot
```

### SSH Host Key Warning After Clone

```bash
# On your local machine:
ssh ubuntu@<clone-ip>
# "WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!"

# Fix: remove the old entry from your known_hosts
ssh-keygen -R <clone-ip>
# Or remove the entire known_hosts (development environments only):
rm ~/.ssh/known_hosts
```

### "virt-sysprep: error: libguestfs appliance failed to launch"

```bash
# Cause: Nested virtualisation or AppArmor restriction
# Fix 1: Use direct backend (bypasses FUSE/QEMU launcher)
export LIBGUESTFS_BACKEND=direct
sudo -E virt-sysprep -a /path/to/disk.qcow2

# Fix 2: Check if KVM is available inside the host
ls -la /dev/kvm

# Fix 3: Check AppArmor
sudo aa-status | grep libguestfs
sudo aa-disable usr.sbin.libvirtd      # Temporary test only
```

### COW Overlay Corruption After Base Modification

```bash
# If the base image was accidentally modified:
# → All overlays are potentially corrupt
# → Do NOT boot any overlay VMs

# Check overlay integrity
qemu-img check /var/lib/libvirt/images/overlay-vm.qcow2

# Flatten the overlay before the base was changed (if you have both versions):
qemu-img convert -f qcow2 -O qcow2 \
  overlay-vm.qcow2 \
  recovered-standalone-vm.qcow2
```

### Clone VM Shows stale cloud-init Data

```bash
# Inside the clone: cloud-init is skipping because it thinks it ran already
cloud-init status
# "status: done" but using old source data

# Fix: clean and re-run
sudo cloud-init clean --logs
sudo cloud-init init --local
sudo cloud-init init
sudo cloud-init modules --mode=config
sudo cloud-init modules --mode=final
```

---

## 15. Full Option Reference Tables

### virt-clone Options

| Option | Description | Example |
|---|---|---|
| `--original` | Source VM name (required) | `--original template-vm` |
| `--name` | New VM name | `--name web-clone-01` |
| `--auto-clone` | Auto-generate name and disk path | `--auto-clone` |
| `--file` | Path for cloned disk image (repeat for multi-disk) | `--file /var/lib/libvirt/images/clone.qcow2` |
| `--skip-copy` | Skip copying a specific disk (target device name) | `--skip-copy vdb` |
| `--mac` | MAC address for the clone NIC | `--mac RANDOM` or `--mac 52:54:00:AA:BB:CC` |
| `--connect` | libvirt connection URI | `--connect qemu:///system` |
| `--preserve-data` | Don't copy disk (reuse source path — dangerous) | `--preserve-data` |
| `--check` | Skip specific safety checks | `--check path_in_use=off` |
| `--print-xml` | Print resulting XML without creating VM | `--print-xml` |

### virt-sysprep Options

| Option | Description | Example |
|---|---|---|
| `-a` | Operate on a disk image file | `-a /path/to/disk.qcow2` |
| `-d` | Operate on a defined VM (must be shut off) | `-d my-vm` |
| `--operations` | Comma-separated list of operations to run | `--operations defaults,ssh-hostkeys` |
| `--no-network` | Disable network access during sysprep | `--no-network` |
| `--hostname` | Set a new hostname | `--hostname template-ubuntu22` |
| `--run-command` | Run a shell command inside the image | `--run-command 'rm -rf /var/lib/cloud'` |
| `--firstboot-command` | Run a command on next VM boot | `--firstboot-command 'cloud-init clean'` |
| `--firstboot-install` | Install packages at first boot | `--firstboot-install qemu-guest-agent` |
| `--ssh-inject` | Inject SSH public key for a user | `--ssh-inject ubuntu:file:~/.ssh/id.pub` |
| `--list-operations` | Show all available operations | `--list-operations` |
| `--dry-run` | Simulate without making changes | `--dry-run` |
| `--verbose` | Show all files being touched | `--verbose` |
| `--exclude-operations` | Exclude specific operations from defaults | `--exclude-operations hostname` |

---

## 16. Quick-Reference Cheat Sheet

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PREFLIGHT (required before ANY clone operation)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 virsh domstate <source-vm>           Must be: shut off
 virsh domblklist <source-vm>         List all disks (need all in virt-clone)
 sudo qemu-img check <disk.qcow2>     Disk must be error-free
 df -h /var/lib/libvirt/images        Enough space for clone?

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PREPARATION (strip unique IDs before cloning)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Automated (recommended):
   sudo virt-sysprep -d <vm>                            Full default cleanup
   sudo virt-sysprep -a <disk.qcow2>                    On image directly
   sudo virt-sysprep -d <vm> --operations defaults \    With Ubuntu extras
     --run-command 'truncate -s 0 /etc/machine-id' \
     --run-command 'rm -f /var/lib/dbus/machine-id' \
     --run-command 'cloud-init clean' \
     --run-command 'rm -rf /var/lib/cloud/'

 Manual (inside the VM before shutting off):
   sudo rm -f /etc/ssh/ssh_host_*                       SSH host keys
   sudo truncate -s 0 /etc/machine-id                   Machine ID
   sudo rm -f /var/lib/dbus/machine-id                  D-Bus ID
   sudo cloud-init clean                                 cloud-init state
   sudo rm -rf /var/lib/cloud/                           cloud-init data
   sudo rm -f /etc/netplan/50-cloud-init.yaml            cloud-init netplan
   sudo pro detach                                       Ubuntu Pro reg
   sudo rm -f /var/lib/dhcp/*.leases                     DHCP state
   sudo shutdown -h now                                  Shut off cleanly

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 CLONING COMMANDS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Auto name/path:
   virt-clone --original <src> --auto-clone

 Named, single disk:
   virt-clone --original <src> --name <new> \
     --file /var/lib/libvirt/images/<new>.qcow2

 Named, two disks:
   virt-clone --original <src> --name <new> \
     --file /path/new-os.qcow2 \
     --file /path/new-data.qcow2

 Skip a disk (share read-only):
   virt-clone --original <src> --name <new> \
     --file /path/new.qcow2 --skip-copy vdb

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 COW OVERLAY CLONING (thin/linked clones)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Create overlay:
   sudo qemu-img create -f qcow2 -b BASE.qcow2 -F qcow2 OVERLAY.qcow2 20G

 Verify chain:
   qemu-img info --backing-chain OVERLAY.qcow2

 Flatten (make independent):
   sudo qemu-img convert -f qcow2 -O qcow2 OVERLAY.qcow2 STANDALONE.qcow2

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 UBUNTU-SPECIFIC UNIQUE IDs TO ALWAYS CLEAR
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 /etc/ssh/ssh_host_*               SSH host keys
 /etc/machine-id                   systemd/DHCP/dbus identity
 /var/lib/dbus/machine-id          D-Bus identity (symlink to above)
 /etc/hostname                     Hostname
 /etc/netplan/50-cloud-init.yaml   cloud-init generated network
 /var/lib/cloud/                   cloud-init all state
 /var/lib/dhcp/*.leases            DHCP lease cache
 /etc/udev/rules.d/70-persistent-net.rules  NIC name persistence

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 POST-CLONE CHECKS (inside the clone)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 hostname                          Should be unique
 cat /etc/machine-id               Should differ from source
 ls /etc/ssh/ssh_host_*            Should exist (regenerated)
 cloud-init status                 Should be "done"
 ip -4 addr show ens3              Should have a unique IP
 sudo pro status                   Re-register if needed

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 KEY UBUNTU vs RHEL DIFFERENCES (cloning context)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Network config    Netplan YAML  (not ifcfg-* scripts)
 Subscription      Ubuntu Pro / pro command  (not RHN/RHSM)
 First-boot        cloud-init  (not firstboot-graphical/initial-setup)
 NIC name          ens3 / enp1s0  (not eth0 by default)
 Udev rules file   Rarely exists on Ubuntu 20.04+
 Key extras        machine-id, cloud-init state, snap IDs
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

> **References:**
> - Ubuntu Server Guide — Virtualisation: https://ubuntu.com/server/docs/virtualization-libvirt
> - libguestfs / virt-sysprep: https://libguestfs.org/virt-sysprep.1.html
> - virt-clone man page: `man virt-clone`
> - cloud-init documentation: https://cloudinit.readthedocs.io/
> - Ubuntu Pro / ua-client: https://ubuntu.com/pro/docs
> - `man virt-sysprep` · `man virt-clone` · `man qemu-img` · `man guestmount`
