# KVM Virtualization — Managing Guest VMs with virsh
## Comprehensive Engineer Reference (Ubuntu / RHEL)

> **Primary Tool:** `virsh` — built on `libvirt` management API  
> **Install:** `sudo apt install libvirt-clients` (Ubuntu) / `yum install libvirt-client` (RHEL)  
> **Terminology:** libvirt uses `domain` = guest virtual machine. In this doc, both terms are used interchangeably.  
> **Non-root access:** virsh works in read-only mode without root; full control requires root or membership in the `libvirt` group.

---

## Table of Contents

1. [Terminology & Key Concepts](#1-terminology--key-concepts)
2. [VM States Reference](#2-vm-states-reference)
3. [Connecting to the Hypervisor](#3-connecting-to-the-hypervisor)
4. [Displaying Version & System Info](#4-displaying-version--system-info)
5. [Listing & Querying VMs](#5-listing--querying-vms)
6. [Starting, Booting & Autostart](#6-starting-booting--autostart)
7. [Stopping, Shutting Down & Destroying](#7-stopping-shutting-down--destroying)
8. [Suspending, Resuming & Resetting](#8-suspending-resuming--resetting)
9. [Saving & Restoring VM State](#9-saving--restoring-vm-state)
10. [Managed Save (Hibernate)](#10-managed-save-hibernate)
11. [Configuration — XML Dump, Define, Edit](#11-configuration--xml-dump-define-edit)
12. [Save-Image Management](#12-save-image-management)
13. [Removing & Deleting VMs](#13-removing--deleting-vms)
14. [VM Information & Statistics](#14-vm-information--statistics)
15. [Block Device Operations](#15-block-device-operations)
16. [Network Interface Operations](#16-network-interface-operations)
17. [Console, Display & Remote Access](#17-console-display--remote-access)
18. [Snapshot Management](#18-snapshot-management)
19. [CPU & vCPU Management](#19-cpu--vcpu-management)
20. [Memory Management](#20-memory-management)
21. [Storage Pool Management](#21-storage-pool-management)
22. [Storage Volume Management](#22-storage-volume-management)
23. [Virtual Network Management](#23-virtual-network-management)
24. [Host / Node Management](#24-host--node-management)
25. [Device Management (Attach/Detach)](#25-device-management-attachdetach)
26. [Host Interface Commands](#26-host-interface-commands)
27. [Resource Tuning (cgroups)](#27-resource-tuning-cgroups)
28. [Miscellaneous Commands](#28-miscellaneous-commands)
29. [QEMU ↔ Domain XML Conversion](#29-qemu--domain-xml-conversion)
30. [CPU Model Configuration](#30-cpu-model-configuration)
31. [Quick Reference Cheat Sheet](#31-quick-reference-cheat-sheet)
32. [Common Troubleshooting Patterns](#32-common-troubleshooting-patterns)

---

## 1. Terminology & Key Concepts

| Term | Meaning |
|------|---------|
| `domain` | libvirt term for a guest virtual machine |
| `guest` / `VM` | Same as domain — used in this doc interchangeably |
| `host` | The physical machine running the hypervisor |
| `hypervisor` | KVM+QEMU — manages guest execution |
| `libvirt` | Management API layer that virsh talks to |
| `libvirtd` | The libvirt daemon running on the host |
| `URI` | Hypervisor connection string (e.g., `qemu:///system`) |
| `UUID` | Universally Unique Identifier — stable across reboots |
| `ID` | Runtime integer ID — changes on every start |
| `Transient VM` | Does NOT survive reboot / shutdown — disappears |
| `Persistent VM` | Survives reboot — has a stored XML definition |

### Naming a VM
You can always refer to a VM by:
- **Name** (short string, e.g., `guest1`) — easiest
- **ID** (integer, e.g., `8`) — only valid while running
- **UUID** (full UUID string) — stable forever

> ⚠️ **User Isolation:** VMs created by `root` are invisible to other users and vice versa. Always use the same user that created the VM. `virt-manager` defaults to root.

### Common Connection URIs

| URI | Meaning |
|-----|---------|
| `qemu:///system` | Connect as root to system-level KVM daemon |
| `qemu:///session` | Connect as current user to their KVM VMs |
| `qemu+ssh://host/system` | Remote system connection over SSH |
| `qemu+tcp://host/system` | Remote connection over TCP (unsecured) |
| `lxc:///` | Connect to local Linux containers |

---

## 2. VM States Reference

### VM Types
- **Transient** — Exists only while running. Shuts down = deleted.
- **Persistent** — Has a stored XML definition. Survives shutdown.

### VM Lifecycle States

| State | Description |
|-------|-------------|
| `undefined` | Not defined — libvirt has no record of it |
| `shut off` | Defined but not running (persistent only) |
| `running` | Active on hypervisor (transient or persistent) |
| `paused` | CPUs suspended; memory still resident; guest unaware |
| `saved` | State saved to disk; qemu process terminated |
| `idle` | Running but waiting on I/O / sleeping |
| `in shutdown` | Received shutdown signal; shutting down gracefully |
| `crashed` | Guest crashed; configured not to restart |
| `pmsuspended` | Suspended by guest power management |

> 💡 When a transient VM is shut off, it ceases to exist entirely.

---

## 3. Connecting to the Hypervisor

### Display Current URI
```bash
virsh uri
# Output: qemu:///session
```

### Connect to Hypervisor
```bash
# System-level (most common — requires root or libvirt group)
virsh connect qemu:///system

# User-level (current user's VMs only)
virsh connect qemu:///session

# Read-only connection
virsh connect qemu:///system --readonly

# Remote host via SSH
virsh connect qemu+ssh://root@remote-host/system
```

> After first `connect`, virsh uses it automatically for all subsequent commands in that session.

### Interactive Shell Mode
```bash
virsh                    # Enter interactive virsh shell
virsh --connect qemu:///system   # Shell with explicit URI
help                     # Show all commands inside virsh shell
help <command>           # Help on a specific command
quit                     # Exit virsh shell
```

---

## 4. Displaying Version & System Info

### virsh Version
```bash
virsh version
# Shows: compiled libvirt version, library in use, API, running hypervisor

virsh version --daemon
# Also shows: libvirtd daemon version and package info
```

Sample output:
```
Compiled against library: libvirt 1.2.8
Using library: libvirt 1.2.8
Using API: QEMU 1.2.8
Running hypervisor: QEMU 1.5.3
Running against daemon: 1.2.8
```

### Echo / Format Testing
```bash
virsh echo "hello world"
virsh echo --shell "value with spaces"   # Outputs shell-safe quoted format
virsh echo --xml "some value"            # Outputs XML-safe format
```

---

## 5. Listing & Querying VMs

### List VMs
```bash
# All VMs (running + stopped)
virsh list --all

# Only running VMs
virsh list

# Only inactive VMs
virsh list --inactive

# Only names (one per line — scriptable)
virsh list --all --name

# Only UUIDs
virsh list --all --uuid

# Table with title/description column
virsh list --all --title

# Persistent only
virsh list --persistent

# Transient only
virsh list --transient

# Only autostarting VMs
virsh list --autostart

# Filter by state
virsh list --state-running
virsh list --state-paused
virsh list --state-shutoff

# VMs with managed save
virsh list --all --with-managed-save

# VMs with snapshots
virsh list --all --with-snapshot
```

Sample output of `virsh list --all`:
```
 Id   Name     State
-----------------------
  8   guest1   running
 22   guest2   paused
  -   guest3   shut off
  -   guest4   shut off
```

### Hypervisor Info
```bash
virsh hostname       # Host's hostname
virsh sysinfo        # Full XML system info (BIOS, manufacturer, etc.)
virsh nodeinfo       # CPU count, memory, topology
virsh capabilities   # Full hypervisor capabilities XML
virsh domcapabilities  # Domain capabilities (VFIO, features, etc.)
```

---

## 6. Starting, Booting & Autostart

### Start a VM
```bash
virsh start guest1

# Start and attach to console immediately
virsh start guest1 --console

# Start in paused state
virsh start guest1 --paused

# Auto-destroy when virsh disconnects
virsh start guest1 --autodestroy

# Skip managed save and force fresh boot
virsh start guest1 --force-boot

# Start restoring from managed save, bypassing file cache
virsh start guest1 --bypass-cache
```

> If the VM was previously saved with `virsh managedsave`, `virsh start` restores from that state automatically unless `--force-boot` is used.

### Create VM from XML (one-shot, transient)
```bash
virsh create guest1.xml            # Start immediately from XML (transient)
```

### Configure Autostart
```bash
# Enable autostart when host boots
virsh autostart guest1

# Disable autostart
virsh autostart guest1 --disable

# Check autostart status
virsh dominfo guest1 | grep -i autostart
```

### Reboot a VM
```bash
virsh reboot guest1

# Specify reboot method
virsh reboot guest1 --mode acpi       # ACPI signal (default)
virsh reboot guest1 --mode agent      # Guest agent
virsh reboot guest1 --mode initctl    # Init system signal
virsh reboot guest1 --mode signal     # Process signal
virsh reboot guest1 --mode paravirt   # Paravirt method

# Multiple modes (driver tries each in undefined order)
virsh reboot guest1 --mode acpi,agent
```

> ℹ️ `virsh reboot` returns once the reboot is *initiated*, not completed. The VM may take additional time.

---

## 7. Stopping, Shutting Down & Destroying

### Graceful Shutdown
```bash
virsh shutdown guest1

# Specify shutdown mode
virsh shutdown guest1 --mode acpi
virsh shutdown guest1 --mode agent
virsh shutdown guest1 --mode initctl
virsh shutdown guest1 --mode signal
virsh shutdown guest1 --mode paravirt
```

> 🔔 Behavior controlled by the `on_shutdown` parameter in VM XML. Changes only take effect after a shutdown+restart cycle.

### Force Kill (Last Resort)
```bash
# Immediate ungraceful shutdown — may corrupt VM filesystems!
virsh destroy guest1

# Attempt to flush disk cache before power off (safer)
virsh destroy guest1 --graceful
```

> ⛔ `virsh destroy` is equivalent to pulling the power cord. Use only when the VM is completely unresponsive. Consider following with `virsh undefine`.

### Comparison Table: Stopping Methods

| Command | Graceful | Saves State | Use When |
|---------|----------|-------------|----------|
| `virsh shutdown` | ✅ Yes | ❌ No | Normal shutdown |
| `virsh destroy` | ❌ No | ❌ No | VM unresponsive |
| `virsh suspend` | N/A | RAM only | Pause briefly |
| `virsh managedsave` | ✅ Yes | ✅ Yes (disk) | Stop + resume later |
| `virsh save` | ✅ Yes | ✅ Yes (disk) | Same as above |

---

## 8. Suspending, Resuming & Resetting

### Suspend (Pause)
```bash
virsh suspend guest1
```
- Consumes RAM but **no CPU, disk, or network I/O**
- Immediate — no graceful shutdown
- Resume with `virsh resume`
- ⚠️ Running on a **transient** VM will **delete** it

### Resume
```bash
virsh resume guest1
```
- Restarts CPUs from exact suspension point
- Only works on persistent VMs
- Does NOT resume a VM that was undefined while suspended

### Reset (Hard Reset)
```bash
virsh reset guest1
```
- Simulates pressing the physical reset button
- Hardware sees RST signal, re-initializes internal state
- **No graceful shutdown** — risk of data loss
- Does NOT apply pending configuration changes (need full shutdown+start for that)

---

## 9. Saving & Restoring VM State

### Save VM State to Disk
```bash
# Save to file (terminates QEMU process, stores RAM to disk)
virsh save guest1 /path/to/guest1.state

# Save with options
virsh save guest1 /path/to/guest1.state --running    # Mark to restore as running
virsh save guest1 /path/to/guest1.state --paused     # Mark to restore as paused
virsh save guest1 /path/to/guest1.state --verbose    # Show progress
virsh save guest1 /path/to/guest1.state --bypass-cache  # Skip FS cache (slower)
virsh save guest1 /path/to/guest1.state --xml alt.xml   # Use alternate XML on restore
```

> 🔑 **save vs suspend:**
> - `virsh suspend` — pauses CPUs, QEMU still runs, memory stays in host RAM (lost on host reboot)
> - `virsh save` — stores memory to disk file, terminates QEMU (survives host reboot)

### Restore VM from Saved State
```bash
virsh restore /path/to/guest1.state

# With options
virsh restore /path/to/guest1.state --running         # Force start as running
virsh restore /path/to/guest1.state --paused          # Force start as paused
virsh restore /path/to/guest1.state --bypass-cache    # Skip FS cache
virsh restore /path/to/guest1.state --xml alt.xml     # Use alternate XML
```

> ℹ️ UUID and name are preserved from save; ID may differ.

### Monitor Save Progress
```bash
virsh domjobinfo guest1    # Check save/restore progress
virsh domjobabort guest1   # Cancel ongoing save/restore
```

---

## 10. Managed Save (Hibernate)

Managed save is virsh-managed state storage — virsh remembers the path automatically.

```bash
# Save and stop VM (hibernates it)
virsh managedsave guest1

# With options
virsh managedsave guest1 --running      # Restore as running
virsh managedsave guest1 --paused       # Restore as paused
virsh managedsave guest1 --bypass-cache # Skip FS cache
virsh managedsave guest1 --verbose      # Show progress

# Restore: just use virsh start — it auto-detects the managed save
virsh start guest1

# Force fresh boot (discard managed save on start)
virsh start guest1 --force-boot

# Remove the managed save file (next start will be fresh boot)
virsh managedsave-remove guest1

# Check if managed save exists
virsh dominfo guest1 | grep "Managed save"
```

> Monitor with `virsh domjobinfo` / cancel with `virsh domjobabort`

---

## 11. Configuration — XML Dump, Define, Edit

### Get VM XML Configuration
```bash
# Dump XML to stdout
virsh dumpxml guest1

# Save XML to file
virsh dumpxml guest1 > guest1.xml

# Include security-sensitive info
virsh dumpxml guest1 --security-info

# Dump inactive config (next-boot config)
virsh dumpxml guest1 --inactive

# Dump with MIGRATABLE flag (for migration compatibility)
virsh dumpxml guest1 --migratable
```

### Create / Define VM from XML
```bash
# Define (register) from XML — does NOT start VM
virsh define guest1.xml

# Create (define + start immediately) — transient
virsh create guest1.xml

# Create with options
virsh create guest1.xml --console   # Attach console after start
virsh create guest1.xml --paused    # Start paused
virsh create guest1.xml --autodestroy  # Destroy when virsh disconnects
```

> 💡 `define` = persistent (survives stop); `create` = transient (destroyed on stop)

### Edit VM XML Live
```bash
virsh edit guest1
# Opens XML in $EDITOR (default: vi)
# Changes take effect after next VM shutdown+start
```

> Use `EDITOR=nano virsh edit guest1` to change the editor for one invocation.

### Validate XML
```bash
virt-xml-validate guest1.xml        # Validate XML without loading
virsh define guest1.xml             # Also validates on load
```

---

## 12. Save-Image Management

> ⚠️ These commands are for **recovery scenarios** — not general use.

### Update XML in a Save Image
```bash
virsh save-image-define /path/to/guest1.state --xml alt.xml
virsh save-image-define /path/to/guest1.state --running   # Mark restore state
virsh save-image-define /path/to/guest1.state --paused
```

### Extract XML from a Save Image
```bash
virsh save-image-dumpxml /path/to/guest1.state
virsh save-image-dumpxml /path/to/guest1.state --security-info
```

### Edit XML in a Save Image
```bash
virsh save-image-edit /path/to/guest1.state
virsh save-image-edit /path/to/guest1.state --running   # Set restore state
virsh save-image-edit /path/to/guest1.state --paused
```

---

## 13. Removing & Deleting VMs

### Undefine (Deregister) a VM
```bash
# Basic undefine
virsh undefine guest1

# Also clean up managed save
virsh undefine guest1 --managed-save

# Also remove snapshot metadata
virsh undefine guest1 --snapshots-metadata

# Remove specific storage volumes with the VM
virsh undefine guest1 --storage vda,vdb

# Remove ALL associated storage volumes (DESTRUCTIVE!)
virsh undefine guest1 --remove-all-storage

# Wipe storage contents before deleting
virsh undefine guest1 --remove-all-storage --wipe-storage

# Remove NVRAM file (for UEFI VMs)
virsh undefine guest1 --nvram
```

> ⚠️ `--remove-all-storage` is irreversible. Only use if no other VMs share the storage.

### Full Delete Workflow
```bash
# Stop + fully delete + remove storage
virsh destroy guest1                              # Force stop if running
virsh undefine guest1 --remove-all-storage       # Remove definition + storage
```

---

## 14. VM Information & Statistics

### General VM Info
```bash
virsh dominfo guest1        # Full info: ID, name, UUID, state, CPU, memory
virsh domid guest1          # Get domain ID (changes each boot)
virsh domname 8             # Get name from ID
virsh domuuid guest1        # Get UUID
virsh domstate guest1       # Current state
virsh domstate guest1 --reason   # State + reason for that state
virsh domcontrol guest1     # Control interface state (ok/error)
virsh domhostname guest1    # Physical host name for this VM
```

### Job Info
```bash
virsh domjobinfo guest1                # Current job status (migration, save, etc.)
virsh domjobinfo guest1 --completed    # Stats of last completed job
virsh domjobabort guest1               # Cancel the running job
```

Sample `domjobinfo` output:
```
Job type:          Unbounded
Time elapsed:      1603 ms
Data processed:    47.004 MiB
Data remaining:    658.633 MiB
Data total:        1.125 GiB
Memory processed:  47.004 MiB
Memory remaining:  658.633 MiB
Memory total:      1.125 GiB
```

### Block Device Info
```bash
# List all block devices for a VM
virsh domblklist guest1
virsh domblklist guest1 --details      # Include type and device info
virsh domblklist guest1 --inactive     # Devices for next boot

# Block device size info
virsh domblkinfo guest1 vda            # By target name
virsh domblkinfo guest1 /path/to/img  # By source path
# Fields: Capacity / Allocation / Physical

# Block device I/O statistics
virsh domblkstat guest1 vda
virsh domblkstat guest1 vda --human    # Human-readable

# Block device errors
virsh domblkerror guest1               # List devices in error state
```

Sample `domblkstat` output:
```
Device: vda
number of read operations:   174670
number of bytes read:        3219440128
number of write operations:  23897
number of bytes written:     164849664
number of flush operations:  11577
```

### Network Interface Info
```bash
# List virtual interfaces
virsh domiflist guest1
virsh domiflist guest1 --inactive      # Interfaces for next boot

# Network I/O statistics
virsh domifstat guest1 macvtap0
virsh domifstat guest1 vnet0
```

Sample `domiflist` output:
```
Interface  Type     Source  Model   MAC
-----------------------------------------------
macvtap0   direct   em1     rtl8139 12:34:00:0f:8a:4a
```

### Memory Statistics
```bash
virsh dommemstat guest1

# With balloon driver collection period
virsh dommemstat guest1 10             # Collect for 10 seconds
virsh dommemstat guest1 --live         # Running stats only
virsh dommemstat guest1 --config       # Persistent config stats
virsh dommemstat guest1 --current      # Current stats
```

Sample `dommemstat` output:
```
actual    1048576
swap_in   0
swap_out  0
major_fault 2974
minor_fault 1272454
unused    246020
available 1011248
rss       865172
```

### CPU Statistics
```bash
virsh cpu-stats guest1             # All CPUs + total
virsh cpu-stats guest1 --total     # Total only
virsh cpu-stats guest1 0 1         # Specific CPU range
```

---

## 15. Block Device Operations

### Block Commit (Shorten Backing Chain)
Copies data from a snapshot down into its backing file.

```bash
# Commit snap2 into snap1 (merges snap2 → snap1)
# Before: base ← snap1 ← snap2 ← active
# After:  base ← snap1 ← active  (snap2 deleted)
virsh blockcommit guest1 disk1 --base snap1 --top snap2 --wait --verbose

# Commit all the way down to the base
virsh blockcommit guest1 disk1 --wait --verbose

# Active commit (while guest runs)
virsh blockcommit guest1 disk1 --active --wait --pivot
```

> ⚠️ `blockcommit` corrupts any file that depends on `--base` other than files depending on `--top`.

### Block Pull (Flatten Backing Chain)
Pulls data from backing files into the active image.

```bash
# Flatten entire chain: base ← sn1 ← sn2 ← active → active (standalone)
virsh blockpull guest1 vda --wait

# Flatten to a specific base: base ← sn1 ← active
virsh blockpull guest1 vda --base /path/to/base.img --wait

# Typical workflow with snapshot first
virsh snapshot-create-as guest1 snap1 --disk-only
virsh blockpull guest1 /path/to/disk --wait
virsh snapshot-delete guest1 snap1 --metadata
```

**Use cases for blockpull:**
- Make a snapshot-based image self-contained
- Flatten part of a chain
- Move disk image to a new filesystem while VM runs
- Post-copy storage migration after live migration

### Resize Block Device (Live)
```bash
# Resize to exact size while VM is running
virsh blockresize guest1 /path/to/disk 50G

# Units: B, KB, KiB, MB, MiB, GB, GiB, TB, TiB
virsh blockresize guest1 vda 90 B

# After resize: guest kernels may auto-detect (virtio-blk)
# SCSI: trigger rescan manually in guest:
echo > /sys/class/scsi_device/0:0:0:0/device/rescan
# IDE: guest reboot required
```

### Disk I/O Throttling
```bash
# Set throttle limits on a block device
virsh blkdeviotune guest1 vda --total-bytes-sec 104857600   # 100 MB/s total
virsh blkdeviotune guest1 vda --read-bytes-sec 52428800     # 50 MB/s read
virsh blkdeviotune guest1 vda --write-bytes-sec 52428800    # 50 MB/s write
virsh blkdeviotune guest1 vda --total-iops-sec 1000
virsh blkdeviotune guest1 vda --read-iops-sec 500
virsh blkdeviotune guest1 vda --write-iops-sec 500

# Apply to running VM
virsh blkdeviotune guest1 vda --read-bytes-sec 52428800 --live

# Persist across reboots
virsh blkdeviotune guest1 vda --read-bytes-sec 52428800 --config

# Query current limits
virsh blkdeviotune guest1 vda
```

### Block I/O Parameters
```bash
# Set I/O weight (cgroups blkio controller)
virsh blkiotune guest1 --weight 500

# Set per-device weights
virsh blkiotune guest1 --device-weights /dev/sda,500

# Query current settings
virsh blkiotune guest1
```

### Trim (Discard Unused Blocks)
```bash
# Trim all mounted filesystems in the guest
virsh domfstrim guest1

# With minimum free range
virsh domfstrim guest1 --minimum 0     # Discard all free blocks

# Specific mount point only
virsh domfstrim guest1 --mountpoint /data
```

---

## 16. Network Interface Operations

### Link State Management
```bash
# Set link state
virsh domif-setlink guest1 vnet0 down
virsh domif-setlink guest1 vnet0 up

# Using MAC address instead of interface name
virsh domif-setlink guest1 52:54:00:01:1d:d0 up

# Persist to config (not just live)
virsh domif-setlink guest1 vnet0 down --config

# Query link state
virsh domif-getlink guest1 vnet0
virsh domif-getlink guest1 52:54:00:01:1d:d0
virsh domif-getlink guest1 vnet0 --config   # Persistent config
```

### Interface Bandwidth (QoS)
```bash
# Query bandwidth settings
virsh domiftune guest1 vnet0

# Set inbound limits (values: average, peak, burst)
# average/peak in KB/s; burst in KB at peak speed
virsh domiftune guest1 eth0 --inbound 1024,2048,512 --live
virsh domiftune guest1 eth0 --outbound 1024,2048,512 --live

# Apply to config (persistent)
virsh domiftune guest1 eth0 --outbound 1024,2048,512 --config

# Clear bandwidth settings
virsh domiftune guest1 eth0 --inbound 0 --outbound 0 --live

# Flags: --config, --live, --current
```

---

## 17. Console, Display & Remote Access

### Serial Console
```bash
virsh console guest1                  # Connect to primary console
virsh console guest1 --safe           # Ensure exclusive access
virsh console guest1 --force          # Disconnect existing sessions
virsh console guest1 --devname serial0  # Specific device alias

# Escape: Ctrl+]  to detach from console
```

> Useful for VMs without VNC/SPICE or network (no SSH) — headless servers.

### Graphical Display URI
```bash
# Get display URI
virsh domdisplay guest1                    # Default display type
virsh domdisplay guest1 --type vnc        # VNC URI
virsh domdisplay guest1 --type spice      # SPICE URI

# Include SPICE channel password
virsh domdisplay guest1 --type spice --include-password
```

### VNC Display Info
```bash
virsh vncdisplay guest1
# Output: 127.0.0.1:0  (IP:port-offset, actual port = 5900+offset)
```

> Requires `graphics type='vnc'` in the domain XML.

### Screenshot
```bash
# Take console screenshot
virsh screenshot guest1

# Save to specific path
virsh screenshot guest1 /home/user/pics/guest1.ppm

# Multiple displays — specify screen ID
virsh screenshot guest1 /tmp/screen.ppm --screen 0
```

---

## 18. Snapshot Management

> ⚠️ **Recommendation:** Use **external** snapshots (not internal) — more flexible, stable, and supported by more tools.

### Create Snapshot
```bash
# Basic (libvirt picks name automatically)
virsh snapshot-create guest1

# From XML file (with specific name, description, disk config)
virsh snapshot-create guest1 --xmlfile snap1.xml

# Disk-only (no memory state — faster, but requires fsck on revert)
virsh snapshot-create guest1 --disk-only

# Disk-only with quiescing (requires guest agent)
virsh snapshot-create guest1 --disk-only --quiesce

# Halt VM after snapshot
virsh snapshot-create guest1 --halt

# Guarantee atomic operation
virsh snapshot-create guest1 --atomic

# External snapshot (RECOMMENDED)
virsh snapshot-create-as guest1 snap1 --diskspec vda,snapshot=external
```

### Create Named Snapshot
```bash
# Basic named snapshot
virsh snapshot-create-as guest1 snap1

# With description
virsh snapshot-create-as guest1 snap1 "Pre-upgrade snapshot"

# Disk-only external snapshot (RECOMMENDED)
virsh snapshot-create-as guest1 snap1 \
  --diskspec vda,snapshot=external,file=/path/to/snap1.qcow2 \
  --disk-only

# Disk-only with quiescing
virsh snapshot-create-as guest1 snap1 --disk-only --quiesce

# Halt after snapshot
virsh snapshot-create-as guest1 snap1 --halt

# Print XML without creating
virsh snapshot-create-as guest1 snap1 --print-xml

# External with specific memory file
virsh snapshot-create-as guest1 snap1 \
  --memspec file=/path/to/snap1.mem,snapshot=external \
  --diskspec vda,snapshot=external,file=/path/to/snap1-disk.qcow2
```

### List Snapshots
```bash
virsh snapshot-list guest1              # All snapshots (name, time, state)
virsh snapshot-list guest1 --parent     # Include parent column
virsh snapshot-list guest1 --roots      # Only top-level (no parent) snapshots
virsh snapshot-list guest1 --tree       # Tree structure view
virsh snapshot-list guest1 --leaves     # Only leaf (no children) snapshots
virsh snapshot-list guest1 --no-leaves  # Only snapshots with children

# Filter by state
virsh snapshot-list guest1 --inactive   # Taken when VM was off
virsh snapshot-list guest1 --active     # Taken while VM was running
virsh snapshot-list guest1 --disk-only  # Disk-only snapshots
virsh snapshot-list guest1 --internal   # Internal snapshots
virsh snapshot-list guest1 --external   # External snapshots

# From a specific snapshot
virsh snapshot-list guest1 --from snap1
virsh snapshot-list guest1 --from snap1 --descendants  # All descendants
virsh snapshot-list guest1 --current --descendants     # From current snapshot
```

### Snapshot Info & XML
```bash
# Current snapshot
virsh snapshot-current guest1
virsh snapshot-current guest1 --name         # Just the name
virsh snapshot-current guest1 --security-info

# Set current snapshot (without reverting)
virsh snapshot-current guest1 snap1

# Get info about specific snapshot
virsh snapshot-info guest1 snap1
virsh snapshot-info guest1 --current

# Dump XML
virsh snapshot-dumpxml guest1 snap1
virsh snapshot-dumpxml guest1 snap1 --security-info

# Get parent of a snapshot
virsh snapshot-parent guest1 snap1
virsh snapshot-parent guest1 --current
```

### Edit Snapshot Metadata
```bash
virsh snapshot-edit guest1 snap1         # Edit in $EDITOR
virsh snapshot-edit guest1 --current     # Edit current snapshot
virsh snapshot-edit guest1 snap1 --rename  # Allow renaming
virsh snapshot-edit guest1 snap1 --clone   # Clone instead of rename
```

### Revert to Snapshot
```bash
virsh snapshot-revert guest1 snap1

# Force-start or force-pause on revert
virsh snapshot-revert guest1 snap1 --running
virsh snapshot-revert guest1 snap1 --paused

# Force revert (when libvirt can't verify config compatibility)
virsh snapshot-revert guest1 snap1 --force
```

> ⚠️ **Destructive!** All changes since the snapshot was taken are **permanently lost**.

### Delete Snapshot
```bash
virsh snapshot-delete guest1 snap1        # Delete single snapshot (merges children)
virsh snapshot-delete guest1 --current    # Delete current snapshot

# Delete with all children
virsh snapshot-delete guest1 snap1 --children

# Delete only children, keep snap1 itself
virsh snapshot-delete guest1 snap1 --children-only

# Delete metadata only (keep actual data)
virsh snapshot-delete guest1 snap1 --metadata
```

---

## 19. CPU & vCPU Management

### Display vCPU Info
```bash
virsh vcpuinfo guest1      # VCPU, host CPU, state, time, affinity

virsh vcpucount guest1     # Count of max/current vCPUs (live + config)
virsh vcpucount guest1 --maximum   # Max vCPU cap
virsh vcpucount guest1 --active    # Currently active vCPUs
virsh vcpucount guest1 --live      # Live values
virsh vcpucount guest1 --config    # Config values (next boot)
virsh vcpucount guest1 --guest     # From guest perspective
```

### vCPU Pinning (CPU Affinity)
```bash
# View current pinning
virsh vcpupin guest1

# Pin vCPU 0 to host CPU 2
virsh vcpupin guest1 0 2

# Pin vCPU 0 to host CPUs 0-3
virsh vcpupin guest1 0 0-3

# Pin vCPU 1 to specific CPUs
virsh vcpupin guest1 1 1,3

# Affect live + persist to config
virsh vcpupin guest1 0 0-3 --live --config
```

### Set Number of vCPUs
```bash
# Hot-add vCPUs while running
virsh setvcpus guest1 4 --live

# Change config for next boot
virsh setvcpus guest1 4 --config

# Both live and config
virsh setvcpus guest1 4 --live --config

# Set maximum vCPUs (only with --config, not --live)
virsh setvcpus guest1 8 --config --maximum

# From guest perspective
virsh setvcpus guest1 4 --guest
```

> ⚠️ Hot-unplug of vCPUs is NOT supported on RHEL 7. Add vCPUs only.

### CPU Model & Capability
```bash
# Show host CPU capabilities
virsh capabilities

# Compare CPU model with host
virsh cpu-compare /path/to/cpu-model.xml

# Compute baseline CPU for a set of hosts
virsh cpu-baseline /path/to/both-cpus.xml

# Emulator CPU info
virsh cpu-models x86_64             # List known CPU models for arch
```

### Emulator Pinning
```bash
# View emulator pinning
virsh emulatorpin guest1

# Pin emulator thread to CPUs
virsh emulatorpin guest1 0-3 --live
virsh emulatorpin guest1 0-3 --config
```

---

## 20. Memory Management

### Set Memory
```bash
# Set current memory (running VM)
virsh setmem guest1 2G --live
virsh setmem guest1 2048M --live
virsh setmem guest1 2097152 --live     # In KiB (default unit)

# Set config for next boot
virsh setmem guest1 2G --config

# Set both live and config
virsh setmem guest1 2G --live --config
```

**Memory unit suffixes:** `b`/`bytes`, `KB`, `k`/`KiB`, `MB`, `M`/`MiB`, `GB`, `G`/`GiB`, `TB`, `T`/`TiB`

> ℹ️ Values are rounded up to nearest KiB. Min ~4000 KiB on most hypervisors.  
> Decreasing memory may cause guest to crash if it's using more than the new limit.

### Set Maximum Memory
```bash
virsh setmaxmem guest1 4G --config     # Set max memory cap (next boot)
virsh setmaxmem guest1 4G --live       # Live (if hypervisor supports)
virsh setmaxmem guest1 4G --current    # Current state
```

### Memory Tuning (cgroups)
```bash
# Set memory limits
virsh memtune guest1 --hard-limit 2G --live
virsh memtune guest1 --soft-limit 1G --live
virsh memtune guest1 --swap-hard-limit 4G --live
virsh memtune guest1 --min-guarantee 512M --live

# Query current settings
virsh memtune guest1
```

### NUMA Memory Parameters
```bash
# View NUMA settings
virsh numatune guest1

# Set NUMA mode and nodeset
virsh numatune guest1 --mode strict --nodeset 0,2-3 --live
virsh numatune guest1 --mode interleave --nodeset 0-1 --live
virsh numatune guest1 --mode preferred --nodeset 0 --live

# Flags: --config, --live, --current
```

---

## 21. Storage Pool Management

### List & Info
```bash
virsh pool-list                   # Active pools only
virsh pool-list --all             # Active + inactive
virsh pool-list --inactive        # Inactive only
virsh pool-list --details         # Extended info
virsh pool-list --autostart       # Autostarting pools
virsh pool-list --type dir        # Filter by type

virsh pool-info vdisk             # Info for specific pool
virsh pool-dumpxml vdisk          # XML config of pool
virsh pool-dumpxml vdisk --inactive  # XML for next boot
```

### Search for Available Pools
```bash
# Find potential sources for a given type
virsh find-storage-pool-sources logical    # Logical (LVM) sources
virsh find-storage-pool-sources netfs      # NFS sources
virsh find-storage-pool-sources iscsi      # iSCSI sources
virsh find-storage-pool-sources disk       # Disk sources

# With host/port hints
virsh find-storage-pool-sources-as disk --host myhost.example.com
```

**Pool types:** `dir`, `fs`, `netfs`, `logical`, `disk`, `iscsi`, `scsi`, `mpath`, `rbd`, `sheepdog`, `gluster`

### Create / Define Pools
```bash
# Method 1: From XML file
# Example XML:
# <pool type="dir">
#   <name>vdisk</name>
#   <target><path>/var/lib/libvirt/images</path></target>
# </pool>

virsh pool-define vdisk.xml       # Define but don't start
virsh pool-create vdisk.xml       # Define and start

# Method 2: From raw parameters
virsh pool-create-as --name vdisk --type dir --target /mnt
virsh pool-define-as --name vdisk --type dir --target /mnt

# Print XML without creating
virsh pool-define-as --name vdisk --type dir --target /mnt --print-xml

# Advanced: NFS pool
virsh pool-define-as --name nfs-pool --type netfs \
  --source-host nfsserver.example.com \
  --source-path /exports/vms \
  --target /mnt/nfs-vms
```

### Start / Stop / Autostart
```bash
virsh pool-start vdisk            # Start a defined (inactive) pool
virsh pool-autostart vdisk        # Enable autostart
virsh pool-autostart vdisk --disable  # Disable autostart
virsh pool-refresh vdisk          # Refresh volume list
virsh pool-destroy vdisk          # Stop pool (data preserved)
virsh pool-delete vdisk           # Destroy underlying storage (IRREVERSIBLE!)
virsh pool-undefine vdisk         # Remove definition (pool becomes transient)
```

### Build Pool (Initialize Storage)
```bash
virsh pool-build vdisk            # Build storage structure
virsh pool-build vdisk --overwrite    # Format device (mkfs), overwrite existing
virsh pool-build vdisk --no-overwrite # Fail if filesystem already exists
```

### Edit Pool Config
```bash
virsh pool-edit vdisk             # Open in $EDITOR (with error checking)
```

---

## 22. Storage Volume Management

### Create Volumes
```bash
# From parameters
virsh vol-create-as vdisk vol-new 100M
virsh vol-create-as vdisk vol-new 100M --format qcow2
virsh vol-create-as vdisk vol-new 100M --format raw
virsh vol-create-as vdisk vol-new 100G --allocation 10G  # Thin-provisioned

# From XML file
virsh vol-create vdisk vol-new.xml

# From another volume (copy)
virsh vol-create-from vdisk vol-new.xml --vol source-vol

# Clone
virsh vol-clone vol-existing vol-new
virsh vol-clone vol-existing vol-new --pool vdisk
```

### List & Info
```bash
virsh vol-list vdisk               # All volumes in a pool
virsh vol-list vdisk --details     # With size info

virsh vol-info vol-new             # Basic info
virsh vol-info vol-new --pool vdisk

virsh vol-dumpxml vol-new          # Full XML
virsh vol-dumpxml vol-new --pool vdisk
```

### Volume Path / Name / Key Lookups
```bash
# Get pool name for a volume (by path)
virsh vol-pool /var/lib/libvirt/images/vol-new

# Get pool UUID
virsh vol-pool /var/lib/libvirt/images/vol-new --uuid

# Get path for a volume
virsh vol-path --pool vdisk vol-new

# Get name from key or path
virsh vol-name /var/lib/libvirt/images/vol-new

# Get key for a volume
virsh vol-key --pool vdisk vol-new
```

### Delete / Wipe Volumes
```bash
# Delete volume
virsh vol-delete vol-new vdisk

# Wipe contents (secure erase) then delete
virsh vol-wipe vol-new vdisk

# Wipe with specific algorithm
virsh vol-wipe vol-new vdisk --algorithm zero       # 1-pass zeroes (default)
virsh vol-wipe vol-new vdisk --algorithm nnsa       # 4-pass NNSA
virsh vol-wipe vol-new vdisk --algorithm dod        # 4-pass DoD
virsh vol-wipe vol-new vdisk --algorithm bsi        # 9-pass BSI
virsh vol-wipe vol-new vdisk --algorithm gutmann    # 35-pass Gutmann
virsh vol-wipe vol-new vdisk --algorithm schneier   # 7-pass Schneier
virsh vol-wipe vol-new vdisk --algorithm random     # 1-pass random
```

### Resize Volume
```bash
virsh vol-resize vol-new 200G --pool vdisk         # Expand to 200G
virsh vol-resize vol-new 200G --pool vdisk --delta # Expand BY 200G
```

---

## 23. Virtual Network Management

### List & Info
```bash
virsh net-list                  # Active networks
virsh net-list --all            # All networks (active + inactive)
virsh net-list --inactive       # Inactive only
virsh net-list --persistent     # Persistent only
virsh net-list --transient      # Transient only
virsh net-list --autostart      # Autostarting only

virsh net-info default          # Info about a network
virsh net-dumpxml default       # Full XML config
virsh net-dumpxml default --inactive  # Inactive config
```

Sample `net-dumpxml` output:
```xml
<network>
  <name>default</name>
  <uuid>98361b46-...</uuid>
  <forward dev='eth0'/>
  <bridge name='virbr0' stp='on' forwardDelay='0'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
```

### Create / Define Networks
```bash
virsh net-create network.xml          # Create + start (transient)
virsh net-define network.xml          # Define only (not started)
virsh net-start default               # Start a defined inactive network
virsh net-autostart default           # Enable autostart
virsh net-autostart default --disable # Disable autostart
```

### Stop / Delete Networks
```bash
virsh net-destroy default             # Stop network
virsh net-undefine default            # Remove definition (make transient)
```

### Edit Network
```bash
virsh net-edit default                # Open in $EDITOR
```

### Conversions
```bash
virsh net-name <UUID>                 # UUID → Name
virsh net-uuid default                # Name → UUID
```

### Set Static IP (DHCP reservation)
```bash
# 1. Find VM's MAC address
virsh domiflist guest1
# Interface  Type    Source  Model   MAC
# vnet4      network default virtio  52:54:00:48:27:1D

# 2. Check DHCP range
virsh net-dumpxml default | grep -E 'range|host mac'

# 3. Add static DHCP reservation
virsh net-update default add ip-dhcp-host \
  '<host mac="52:54:00:48:27:1D" ip="192.168.122.100"/>' \
  --live --config

# 4. Bounce the interface to get new IP
virsh domif-setlink guest1 52:54:00:48:27:1D down
sleep 10
virsh domif-setlink guest1 52:54:00:48:27:1D up
```

### Update Network Config (Live)
```bash
# Add DHCP host entry
virsh net-update default add ip-dhcp-host '<host mac="..." ip="..."/>' --live --config

# Delete DHCP host entry
virsh net-update default delete ip-dhcp-host '<host mac="..."/>' --live --config

# Modify
virsh net-update default modify ip-dhcp-host '<host mac="..." ip="..."/>' --live --config

# Add DNS entry
virsh net-update default add dns-host '<host ip="1.2.3.4"><hostname>myhost</hostname></host>' --live --config

# Sections: bridge, domain, ip, ip-dhcp-host, ip-dhcp-range,
#           forward, forward-interface, portgroup, dns-host, dns-txt, dns-srv
```

---

## 24. Host / Node Management

### Host Info
```bash
virsh nodeinfo            # CPU model, count, frequency, memory, NUMA topology
virsh hostname            # Host machine hostname
virsh sysinfo             # Full XML BIOS/system info

# Sample nodeinfo output:
# CPU model: x86_64
# CPU(s): 4
# CPU frequency: 2400 MHz
# CPU socket(s): 1
# Core(s) per socket: 2
# Thread(s) per core: 2
# NUMA cell(s): 1
# Memory size: 8192000 KiB
```

### CPU Statistics
```bash
virsh nodecpumap               # Number of CPUs present and online
virsh nodecpustats             # CPU load statistics (all CPUs)
virsh nodecpustats 0           # Specific CPU stats
virsh nodecpustats --percent   # As percentages over 1 second
virsh nodecpustats 2 --percent # CPU 2 as percentages
```

### Memory / NUMA
```bash
# Free memory in NUMA cells
virsh freecell              # Total free memory
virsh freecell --all        # Per-cell breakdown
virsh freecell --cellno 0   # Specific cell

# KSM (Kernel Same-page Merging) settings
virsh node-memory-tune --shm-pages-to-scan 100
virsh node-memory-tune --shm-sleep-milisecs 10
virsh node-memory-tune --shm-merge-across-nodes 1  # Merge across NUMA nodes
virsh node-memory-tune                              # Query current settings
```

### Host Devices
```bash
# List all host devices
virsh nodedev-list
virsh nodedev-list --tree              # Tree view
virsh nodedev-list --cap scsi         # Filter by capability
virsh nodedev-list --cap pci          # PCI devices
virsh nodedev-list --cap net          # Network devices
virsh nodedev-list --cap usb_device   # USB devices

# Device XML
virsh nodedev-dumpxml pci_0000_00_02_0  # Full XML for device

# Create device (register with libvirt)
virsh nodedev-create device.xml

# Remove device (detach from host — for passthrough)
virsh nodedev-destroy pci_0000_00_02_0
virsh nodedev-destroy pci_0000_00_02_0 --driver vfio  # Specify driver

# Reset a device
virsh nodedev-reset pci_0000_00_02_0
```

---

## 25. Device Management (Attach/Detach)

### Attach/Detach Generic Device
```bash
# Attach from XML file
virsh attach-device guest1 --file device.xml --live     # Live only
virsh attach-device guest1 --file device.xml --config   # Next boot
virsh attach-device guest1 --file device.xml --current  # Current state

# Detach
virsh detach-device guest1 --file device.xml
virsh detach-device guest1 --file device.xml --live
virsh detach-device guest1 --file device.xml --config
```

### Hot-Plug USB Device
```bash
# 1. Find device
lsusb -v | grep -E "idVendor|idProduct"

# 2. Create XML (usb_device.xml):
# <hostdev mode='subsystem' type='usb'>
#   <source>
#     <vendor id='0x17ef'/>
#     <product id='0x480f'/>
#   </source>
# </hostdev>

# 3. Attach
virsh attach-device guest1 --file usb_device.xml --current

# 4. Detach
virsh detach-device guest1 --file usb_device.xml
```

### Attach/Detach Network Interface
```bash
# Attach network interface
virsh attach-interface guest1 network default --model virtio --live

# Attach with specific MAC
virsh attach-interface guest1 network default \
  --model virtio --mac 52:54:00:aa:bb:cc --live

# Attach bridge interface
virsh attach-interface guest1 bridge br0 --model virtio

# With bandwidth QoS
virsh attach-interface guest1 network default \
  --model virtio \
  --inbound average=1024,peak=2048,burst=512 \
  --outbound average=1024,peak=2048,burst=512

# Detach by MAC
virsh detach-interface guest1 network --mac 52:54:00:aa:bb:cc --live
```

### Attach/Detach Disk
```bash
# Attach disk
virsh attach-disk guest1 /path/to/disk.img vdb --live

# Attach with specific driver/type
virsh attach-disk guest1 /path/to/disk.qcow2 vdb \
  --driver qemu --subdriver qcow2 --live

# Detach disk
virsh detach-disk guest1 vdb --live

# Attach as read-only
virsh attach-disk guest1 /path/to/iso sdc --type cdrom \
  --mode readonly --live
```

### Change CDROM Media
```bash
virsh change-media guest1 hdc /path/to/new.iso --live   # Insert new media
virsh change-media guest1 hdc --eject --live             # Eject media
virsh change-media guest1 hdc /path/to/new.iso --update --live
virsh change-media guest1 hdc /path/to/new.iso --force   # Force change
```

---

## 26. Host Interface Commands

> ⚠️ Run from the **host machine terminal**, NOT from inside the guest.  
> ⚠️ Requires **network service** (not NetworkManager) to be active.

```bash
# Define interface from XML
virsh iface-define eth4.xml

# List interfaces
virsh iface-list            # Active only
virsh iface-list --all      # Active + inactive
virsh iface-list --inactive # Inactive only

# Start/stop interface
virsh iface-start eth4
virsh iface-destroy eth4    # Stop (same as if-down)

# MAC ↔ Name conversion
virsh iface-name 00:11:22:33:44:55    # MAC → Name
virsh iface-mac eth0                   # Name → MAC

# Undefine interface
virsh iface-undefine eth4

# Show interface XML
virsh iface-dumpxml eth0
virsh iface-dumpxml eth0 --inactive    # Next-boot config

# Edit interface XML
virsh iface-edit eth0
```

### Bridge Management
```bash
# Create bridge and attach interface
virsh iface-bridge eth0 br0             # Create br0 from eth0
virsh iface-bridge eth0 br0 --no-stp   # Disable STP
virsh iface-bridge eth0 br0 delay=5    # Set STP delay
virsh iface-bridge eth0 br0 --no-start # Don't start immediately

# Remove bridge
virsh iface-unbridge br0               # Returns eth0 to normal
virsh iface-unbridge br0 --no-start    # Don't restart underlying iface
```

### Interface Snapshots (for safe changes)
```bash
# Workflow for safe interface changes
virsh iface-begin              # Snapshot current config

virsh iface-define eth4.xml    # Make changes
virsh iface-start eth4

# If something breaks:
virsh iface-rollback           # Revert to snapshot

# If everything works:
virsh iface-commit             # Commit changes

# NOTE: Automatic rollback occurs on host reboot before commit
```

---

## 27. Resource Tuning (cgroups)

libvirt manages cgroups for VMs automatically. These commands adjust the cgroup parameters.

### CPU Scheduling
```bash
# View schedule parameters
virsh schedinfo guest1

# Set cpu_shares (relative CPU weight, default 1024)
virsh schedinfo guest1 --set cpu_shares=2048 --live

# Set CFS quota (microseconds per period; -1 = unlimited)
virsh schedinfo guest1 --set vcpu_period=100000 --live
virsh schedinfo guest1 --set vcpu_quota=50000 --live  # 50% of 1 CPU

# Emulator thread tuning
virsh schedinfo guest1 --set emulator_period=100000 --live
virsh schedinfo guest1 --set emulator_quota=25000 --live

# Flags: --current, --config, --live
```

**cgroups → virsh parameter mapping:**

| cgroup field | virsh parameter |
|---|---|
| `cpu.shares` | `cpu_shares` |
| `cpu.cfs_period_us` | `vcpu_period` |
| `cpu.cfs_quota_us` | `vcpu_quota` |

### Disk I/O Throttling
```bash
# Set per-device throttle
virsh blkdeviotune guest1 vda \
  --total-bytes-sec 104857600 \
  --total-iops-sec 1000 --live

# Query
virsh blkdeviotune guest1 vda
```

### Block I/O Weight
```bash
virsh blkiotune guest1 --weight 500             # VM-wide I/O weight
virsh blkiotune guest1 --device-weights /dev/sda,500
```

### Network Bandwidth
```bash
virsh domiftune guest1 vnet0 --inbound 1024,2048,512 --live
virsh domiftune guest1 vnet0 --outbound 1024,2048,512 --live
```

### Memory Limits
```bash
virsh memtune guest1 --hard-limit 4G
virsh memtune guest1 --soft-limit 3G
virsh memtune guest1 --swap-hard-limit 8G
virsh memtune guest1 --min-guarantee 1G
```

---

## 28. Miscellaneous Commands

### Inject NMI
```bash
# Send Non-Maskable Interrupt (for panic/crashdump)
virsh inject-nmi guest1

# Use cases:
# - Trigger kernel panic for crashdump collection
# - Debug non-recoverable hardware errors
# - Trigger crashdump in Windows guests
```

### Send Keystrokes
```bash
# Send Ctrl+Alt+Del
virsh send-key guest1 --codeset linux --holdtime 1000 \
  KEY_LEFTCTRL KEY_LEFTALT KEY_DELETE

# Send individual key
virsh send-key guest1 --codeset linux KEY_ENTER

# Code sets: linux, xt, atset1, atset2, atset3, os_x, xt_kbd, win32, usb, rfb
```

> ⚠️ Multiple keycodes are sent simultaneously (random order). For sequential input, run `send-key` multiple times.

### Create Core Dump
```bash
# Basic dump
virsh dump guest1 /path/to/core.dump

# Memory-only dump (REQUIRED for crash utility)
virsh dump guest1 /path/to/core.dump --memory-only

# Memory-only, reset after dump
virsh dump guest1 /path/to/core.dump --memory-only --reset

# Live dump (don't pause guest)
virsh dump guest1 /path/to/core.dump --live

# Crash guest (mark as crashed)
virsh dump guest1 /path/to/core.dump --crash

# With compression format
virsh dump guest1 /path/to/core.dump --memory-only --format elf
virsh dump guest1 /path/to/core.dump --memory-only --format kdump-zlib
virsh dump guest1 /path/to/core.dump --memory-only --format kdump-lzo
virsh dump guest1 /path/to/core.dump --memory-only --format kdump-snappy

# Show progress
virsh dump guest1 /path/to/core.dump --verbose

# Monitor/cancel
virsh domjobinfo guest1
virsh domjobabort guest1
```

> ⚠️ The `crash` utility requires `--memory-only`. Also use `--memory-only` for Red Hat support cases.

### Get Domain Capabilities
```bash
virsh domcapabilities          # Full capabilities XML
virsh domcapabilities --arch x86_64
virsh domcapabilities --machine q35

# Check VFIO/IOMMU support
virsh domcapabilities | grep -A5 iommu
```

---

## 29. QEMU ↔ Domain XML Conversion

Convert existing QEMU command-line args to libvirt domain XML.

```bash
# From a QEMU args file
virsh domxml-from-native qemu-kvm demo.args

# Save to file
virsh domxml-from-native qemu-kvm demo.args > guest1.xml

# Then register with libvirt
virsh define guest1.xml
```

Example args file (`demo.args`):
```
LC_ALL=C /usr/bin/qemu -S -M pc -m 2048 -smp 2 -nographic \
  -hda /dev/HostVG/QEMUGuest1 -net none -serial none
```

> ℹ️ Use this **only** to migrate existing QEMU guests into libvirt management. Create new VMs with `virsh`, `virt-install`, or `virt-manager`.

---

## 30. CPU Model Configuration

### Discover Host CPU Model
```bash
virsh capabilities | grep -A 20 '<cpu>'
```

### Compare CPU Compatibility
```bash
# Check if host CPU is superset of desired model
virsh cpu-compare desired-cpu.xml
# Output: "Host CPU is a superset of..." = compatible
# Output: "CPUs are not strictly compatible" = issues exist
```

### Compute Baseline CPU (for Migration Pools)
```bash
# Given XML files for multiple hosts, find common CPU
virsh cpu-baseline both-hosts.xml

# Use output directly in guest XML under <domain>
```

### CPU Match Policies

| `match=` value | Behavior |
|---|---|
| `minimum` | Host must have at least these features; extras are exposed too |
| `exact` | Host must have at least these features; extras are **masked** |
| `strict` | Host must have **exactly** these features |

### Feature Policies

| `policy=` value | Behavior |
|---|---|
| `force` | Expose even if host lacks it (software emulation) |
| `require` | Expose and fail if host lacks it (safe default) |
| `optional` | Expose if available; becomes `require` on live migration |
| `disable` | Hide from guest even if host has it |
| `forbid` | Fail to start if host has this feature |

---

## 31. Quick Reference Cheat Sheet

### VM Lifecycle
```bash
virsh list --all                          # See all VMs
virsh start guest1                        # Start VM
virsh shutdown guest1                     # Graceful stop
virsh destroy guest1                      # Force kill
virsh reboot guest1                       # Reboot
virsh suspend guest1                      # Pause (RAM)
virsh resume guest1                       # Unpause
virsh reset guest1                        # Hard reset
virsh managedsave guest1                  # Hibernate to disk
virsh managedsave-remove guest1          # Remove hibernate image
virsh autostart guest1                    # Enable autostart
virsh autostart guest1 --disable          # Disable autostart
```

### VM Config
```bash
virsh dumpxml guest1                      # Get XML
virsh dumpxml guest1 > guest1.xml        # Save XML
virsh define guest1.xml                   # Register VM from XML
virsh create guest1.xml                   # Start VM from XML (transient)
virsh edit guest1                         # Edit XML in $EDITOR
virsh undefine guest1                     # Remove VM definition
virsh undefine guest1 --remove-all-storage  # Remove VM + storage
```

### VM Info
```bash
virsh dominfo guest1                      # General info
virsh domstate guest1                     # State
virsh domid guest1                        # Runtime ID
virsh domname 8                           # Name from ID
virsh domuuid guest1                      # UUID
virsh vcpuinfo guest1                     # vCPU info
virsh dommemstat guest1                   # Memory stats
virsh domblklist guest1                   # Block devices
virsh domiflist guest1                    # Network interfaces
virsh domblkstat guest1 vda              # Block I/O stats
virsh domifstat guest1 vnet0             # Net I/O stats
virsh cpu-stats guest1                   # CPU stats
```

### Snapshots
```bash
virsh snapshot-create-as guest1 snap1 --disk-only  # Create disk snapshot
virsh snapshot-list guest1                          # List snapshots
virsh snapshot-revert guest1 snap1                  # Revert to snapshot
virsh snapshot-delete guest1 snap1                  # Delete snapshot
virsh snapshot-info guest1 snap1                    # Snapshot details
```

### Storage
```bash
virsh pool-list --all                     # List storage pools
virsh pool-create-as --name vdisk --type dir --target /mnt  # Create pool
virsh pool-start vdisk                    # Start pool
virsh pool-destroy vdisk                  # Stop pool
virsh vol-list vdisk                      # List volumes in pool
virsh vol-create-as vdisk new-vol 10G --format qcow2  # Create volume
virsh vol-delete new-vol vdisk            # Delete volume
```

### Networking
```bash
virsh net-list --all                      # List virtual networks
virsh net-dumpxml default                 # Network XML
virsh net-start default                   # Start network
virsh net-destroy default                 # Stop network
virsh net-edit default                    # Edit network
```

### Host
```bash
virsh nodeinfo                            # Host info
virsh hostname                            # Host name
virsh capabilities                        # Hypervisor capabilities
virsh nodedev-list --tree                 # Host device tree
virsh freecell --all                      # NUMA free memory
```

---

## 32. Common Troubleshooting Patterns

### VM Won't List
```bash
# Check you're using the right user
virsh list --all        # If empty despite knowing VMs exist:

# Try system-level connection
virsh --connect qemu:///system list --all

# Try session-level
virsh --connect qemu:///session list --all

# Check libvirtd is running
systemctl status libvirtd
```

### VM Stuck / Unresponsive
```bash
# Check state + reason
virsh domstate guest1 --reason

# Check for block device errors
virsh domblkerror guest1

# Force kill
virsh destroy guest1

# Check domain control interface
virsh domcontrol guest1     # Should be 'ok'
```

### Can't Undefine VM
```bash
# If VM has managed save:
virsh undefine guest1 --managed-save

# If VM has snapshots:
virsh undefine guest1 --snapshots-metadata

# If VM is still running, destroy first:
virsh destroy guest1
virsh undefine guest1
```

### Console Won't Connect
```bash
# Ensure serial console is configured in VM XML:
# <serial type='pty'><target port='0'/></serial>
# <console type='pty'><target type='serial' port='0'/></console>

# Inside VM, ensure getty on serial:
# systemctl enable serial-getty@ttyS0.service

virsh console guest1 --force    # Force disconnect existing sessions
```

### Check Job Progress
```bash
# Monitor ongoing save/migrate/dump
watch -n1 virsh domjobinfo guest1

# Cancel if needed
virsh domjobabort guest1
```

### VM Uses Wrong CPU Feature at Migration
```bash
# Compare guest CPU config with destination host
virsh cpu-compare /path/to/guest-cpu.xml

# Compute safe baseline for migration pool
virsh cpu-baseline /path/to/both-hosts.xml

# Copy output into guest XML <domain> element
virsh edit guest1
```

### Disk Full / Performance Issues
```bash
# Check block device size and allocation
virsh domblkinfo guest1 vda

# Check I/O throttle limits
virsh blkdeviotune guest1 vda

# Resize disk (online)
virsh blockresize guest1 vda 100G

# Trim unused blocks in guest
virsh domfstrim guest1 --minimum 0
```

### Network Interface Down
```bash
# Check link state
virsh domif-getlink guest1 vnet0

# Bring up
virsh domif-setlink guest1 vnet0 up

# Check stats for errors
virsh domifstat guest1 vnet0
```

---

## Appendix A — Important Flags Reference

| Flag | Meaning |
|------|---------|
| `--live` | Affects running VM immediately |
| `--config` | Affects stored XML (next boot) |
| `--current` | Affects current state (running or config) |
| `--all` | Include both active and inactive |
| `--inactive` | Only inactive items |
| `--verbose` | Show progress |
| `--bypass-cache` | Avoid filesystem cache (may be slower) |
| `--force` | Force the operation |
| `--wait` | Wait for async operation to complete |
| `--pivot` | Pivot disk after blockcommit/blockpull |

## Appendix B — File Locations

| Path | Description |
|------|-------------|
| `/etc/libvirt/qemu/` | VM XML definitions |
| `/var/lib/libvirt/images/` | Default storage pool |
| `/var/lib/libvirt/qemu/save/` | Managed save files |
| `/var/lib/libvirt/qemu/snapshot/` | Snapshot metadata |
| `/var/run/libvirt/qemu/` | Runtime PID, monitor sockets |
| `/var/log/libvirt/qemu/` | Per-VM QEMU logs |
| `/etc/libvirt/libvirtd.conf` | libvirt daemon config |
| `/etc/libvirt/qemu.conf` | QEMU driver config |

## Appendix C — virsh Help

```bash
virsh help                  # All command groups
virsh help domain           # Domain management commands
virsh help storage          # Storage commands
virsh help network          # Network commands
virsh help host             # Host commands
virsh help snapshot         # Snapshot commands
virsh help <command>        # Help on specific command
man virsh                   # Full manual page
```

---

*Reference: libvirt upstream documentation — https://libvirt.org/docs.html*  
*virsh man page: `man virsh`*
