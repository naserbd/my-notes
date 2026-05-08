# KVM on Ubuntu — Network Configuration
### Engineer's Extensive Reference | Topic: KVM Networking Only
> Ubuntu 20.04 / 22.04 / 24.04 LTS · libvirt · Netplan · Open vSwitch · SR-IOV  
> Scope: NAT → Bridging → Custom networks → VLANs → SR-IOV → OVS → vhost-net → Troubleshooting

---

## Table of Contents

1. [KVM Networking Architecture Overview](#1-kvm-networking-architecture-overview)
2. [Supported Network Types — Summary](#2-supported-network-types--summary)
3. [Preflight — Network Prerequisites](#3-preflight--network-prerequisites)
4. [NAT Networking (Default Virtual Network)](#4-nat-networking-default-virtual-network)
5. [Custom NAT / Isolated Networks](#5-custom-nat--isolated-networks)
6. [Bridged Networking (Ubuntu / Netplan)](#6-bridged-networking-ubuntu--netplan)
7. [Routed Networking](#7-routed-networking)
8. [Isolated / Private Networking (No External Access)](#8-isolated--private-networking-no-external-access)
9. [VLAN Networking](#9-vlan-networking)
10. [Open vSwitch (OVS) Networking](#10-open-vswitch-ovs-networking)
11. [PCI Device Assignment (Passthrough)](#11-pci-device-assignment-passthrough)
12. [SR-IOV (PCIe Virtual Functions)](#12-sr-iov-pcie-virtual-functions)
13. [vhost-net — Kernel-Level Virtio Backend](#13-vhost-net--kernel-level-virtio-backend)
14. [Macvtap / macvlan Networking](#14-macvtap--macvlan-networking)
15. [Guest XML Network Interface Configuration](#15-guest-xml-network-interface-configuration)
16. [UFW Firewall and libvirt](#16-ufw-firewall-and-libvirt)
17. [IP Forwarding and Routing](#17-ip-forwarding-and-routing)
18. [DNS and DHCP Inside Virtual Networks](#18-dns-and-dhcp-inside-virtual-networks)
19. [Network Performance Tuning](#19-network-performance-tuning)
20. [Monitoring and Diagnostics](#20-monitoring-and-diagnostics)
21. [Troubleshooting Quick Reference](#21-troubleshooting-quick-reference)
22. [Quick-Reference Cheat Sheet](#22-quick-reference-cheat-sheet)

---

## 1. KVM Networking Architecture Overview

### 1.1 How VM Networking Works

```
┌─────────────────────────────────────────────────────────────────┐
│                        Ubuntu KVM Host                          │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Guest VM1  │  │   Guest VM2  │  │   Guest VM3  │          │
│  │  eth0/ens3   │  │  eth0/ens3   │  │  eth0/ens3   │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                 │                   │
│  ┌──────▼─────────────────▼─────────────────▼───────┐          │
│  │    Virtual Switch (virbr0 / br0 / ovs-br0)       │          │
│  │    [libvirt network / Linux bridge / OVS]         │          │
│  └──────────────────────────┬────────────────────────┘          │
│                             │                                   │
│  ┌──────────────────────────▼────────────────────────┐          │
│  │          Host Network Stack                        │          │
│  │  iptables/nftables · ip_forward · routing table   │          │
│  └──────────────────────────┬────────────────────────┘          │
│                             │                                   │
│  ┌──────────────────────────▼────────────────────────┐          │
│  │        Physical NIC (enp3s0 / ens3 / bond0)       │          │
│  └───────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                         Physical LAN
```

### 1.2 Virtual NIC (vNIC) to Host Mapping

Each guest NIC appears as a **tap device** (`vnetX`) on the host. The tap device is enslaved to a bridge or managed by a network backend.

```bash
# View tap devices created for running VMs
ip link show type tun
# or
ip link show | grep vnet

# Sample output:
# vnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...   ← VM1's NIC
# vnet1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...   ← VM2's NIC

# See which bridge each tap belongs to
brctl show
# bridge name   bridge id         STP enabled   interfaces
# virbr0        8000.525400000001 yes           vnet0
#                                               vnet1
# br0           8000.AABBCCDDEEFF no            enp3s0
#                                               vnet2
```

### 1.3 Key Networking Components on Ubuntu

| Component | Package | Role |
|---|---|---|
| `libvirtd` | `libvirt-daemon-system` | Manages virtual networks, DHCP, DNS |
| `dnsmasq` | Auto-installed by libvirt | DHCP + DNS server for NAT networks |
| `iptables`/`nftables` | Built-in | NAT rules, forwarding rules |
| `bridge-utils` | `bridge-utils` | `brctl` tool for bridge management |
| `netplan` | Built-in on Ubuntu | Network config: bridges, VLANs, bonds |
| `openvswitch-switch` | `openvswitch-switch` | OVS for advanced SDN networking |
| `vhost-net` | Kernel module | In-kernel virtio packet processing |

---

## 2. Supported Network Types — Summary

| Type | Guest IP from | LAN visible? | Inbound from LAN? | Live migration? | Use case |
|---|---|---|---|---|---|
| **NAT** | libvirt dnsmasq | No | No (without port-forward) | No | Dev/test, outbound-only |
| **Bridge** | LAN DHCP or static | Yes | Yes | Yes | Production, servers |
| **Routed** | libvirt dnsmasq | Routed | If routed | No | Multi-subnet routing |
| **Isolated** | libvirt dnsmasq | No | No | No | Air-gapped, internal |
| **Macvtap** | LAN DHCP | Yes | Yes | Limited | Simple LAN access |
| **PCI passthrough** | Physical NIC | Yes | Yes | No | High-perf, bare metal |
| **SR-IOV** | Physical NIC VF | Yes | Yes | Limited | High-perf, scalable |
| **OVS** | Managed by OVS | Configurable | Configurable | Yes | SDN, cloud, OpenStack |
| **No network** | None | No | No | N/A | Isolated security VMs |

---

## 3. Preflight — Network Prerequisites

```bash
# 1. Install all networking-related packages
sudo apt update && sudo apt install -y \
  libvirt-daemon-system \
  libvirt-clients \
  bridge-utils \
  virtinst \
  openvswitch-switch \     # For OVS (optional)
  vlan \                   # For VLAN support
  net-tools \              # ifconfig, netstat, route
  iproute2 \               # ip, ss, tc commands
  ethtool \                # NIC feature inspection
  tcpdump \                # Packet capture
  nmap                     # Network scanning/testing

# 2. Enable IP forwarding (required for NAT and routed networks)
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-kvm-forward.conf
echo "net.ipv6.conf.all.forwarding = 1" | sudo tee -a /etc/sysctl.d/99-kvm-forward.conf
sudo sysctl -p /etc/sysctl.d/99-kvm-forward.conf

# Verify
sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 1  ← required

# 3. Load required kernel modules
sudo modprobe bridge
sudo modprobe 8021q                    # VLAN support
sudo modprobe vhost_net                # Kernel-level virtio backend
lsmod | grep -E 'bridge|vhost_net|8021q'

# Persist across reboots
echo -e "bridge\n8021q\nvhost_net" | sudo tee /etc/modules-load.d/kvm-networking.conf

# 4. Verify libvirtd is running
sudo systemctl status libvirtd        # Ubuntu 20.04
sudo systemctl status virtqemud       # Ubuntu 22.04+

# 5. Verify default network exists
virsh net-list --all
# NAME      STATE    AUTOSTART   PERSISTENT
# default   active   yes         yes

# 6. Find your physical NIC name
ip link show
# Use the name that has the host's outbound route
ip route get 8.8.8.8 | awk '/dev/{print $5}'
```

---

## 4. NAT Networking (Default Virtual Network)

### 4.1 How NAT Works with libvirt

```
Guest VM (192.168.122.x)
         │
    tap device (vnet0) — enslaved to virbr0
         │
    virbr0 bridge (192.168.122.1) — host-only bridge, no physical NIC attached
         │
    iptables MASQUERADE rule
         │
    Host physical NIC (enp3s0) → Internet/LAN
```

libvirt manages:
- The `virbr0` bridge device
- A `dnsmasq` process for DHCP (192.168.122.2–254 by default) and DNS
- iptables/nftables rules for NAT (MASQUERADE) and forwarding

### 4.2 Verify the Default Network

```bash
# List all virtual networks
virsh net-list --all

# Detailed network information
virsh net-info default

# Full XML definition
virsh net-dumpxml default

# Check the bridge on the host
brctl show virbr0
ip addr show virbr0     # Should show 192.168.122.1/24

# Check dnsmasq is running for the network
ps aux | grep dnsmasq
# /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf ...

# Check NAT rules libvirt added
sudo iptables -t nat -L -n -v | grep -E 'virbr|MASQUERADE'
# MASQUERADE  all  --  192.168.122.0/24   !192.168.122.0/24
```

### 4.3 Start, Stop, and Autostart the Default Network

```bash
# Start
sudo virsh net-start default

# Stop
sudo virsh net-destroy default

# Set to auto-start on host boot
sudo virsh net-autostart default
sudo virsh net-autostart default --disable     # Disable autostart

# Verify autostart
virsh net-info default | grep Autostart
```

### 4.4 Recreate the Default Network (If Missing)

The default network definition file is at `/usr/share/libvirt/networks/default.xml` on Ubuntu.

```bash
# Check if the default network definition exists
ls /etc/libvirt/qemu/networks/
cat /etc/libvirt/qemu/networks/default.xml 2>/dev/null || echo "Not defined"

# If missing, create it
cat > /tmp/default-network.xml <<'EOF'
<network>
  <name>default</name>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr0' stp='on' delay='0'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
EOF

sudo virsh net-define /tmp/default-network.xml
sudo virsh net-start default
sudo virsh net-autostart default
virsh net-list --all
```

### 4.5 Connect a Guest to the Default NAT Network

**In the guest XML (virsh edit \<vmname\>):**

```xml
<!-- Minimal — libvirt assigns MAC automatically -->
<interface type='network'>
  <source network='default'/>
  <model type='virtio'/>
</interface>

<!-- With an explicit MAC address -->
<interface type='network'>
  <source network='default'/>
  <model type='virtio'/>
  <mac address='52:54:00:1a:b3:4a'/>
</interface>
```

**At creation time (virt-install):**

```bash
# Default (NAT is used automatically when no --network is specified)
virt-install ... --network network=default,model=virtio

# Or omit --network entirely — same result
```

### 4.6 DHCP Lease Management

```bash
# View all current DHCP leases on the default network
virsh net-dhcp-leases default
# Expiry-Time   MAC address          Protocol   IP address    Hostname

# Static DHCP assignment (always same IP for a specific MAC)
virsh net-update default add ip-dhcp-host \
  "<host mac='52:54:00:aa:bb:cc' name='web01' ip='192.168.122.100'/>" \
  --live --config

# Remove a static assignment
virsh net-update default delete ip-dhcp-host \
  "<host mac='52:54:00:aa:bb:cc'/>" \
  --live --config

# List static hosts
virsh net-dumpxml default | grep '<host'

# Directly view the dnsmasq lease file
cat /var/lib/libvirt/dnsmasq/default.leases
```

### 4.7 Port Forwarding into NAT Guests

NAT guests are not directly reachable from outside. Add iptables rules to forward specific ports:

```bash
# Forward host port 8080 → guest 192.168.122.100:80
GUEST_IP="192.168.122.100"
HOST_PORT=8080
GUEST_PORT=80
HOST_IF="enp3s0"

# DNAT rule: incoming on host → redirect to guest
sudo iptables -t nat -I PREROUTING -i "$HOST_IF" \
  -p tcp --dport $HOST_PORT \
  -j DNAT --to-destination "${GUEST_IP}:${GUEST_PORT}"

# Allow the forwarded traffic through the bridge
sudo iptables -I FORWARD -d "$GUEST_IP" \
  -p tcp --dport $GUEST_PORT \
  -j ACCEPT

# Make persistent across reboots
sudo apt install iptables-persistent
sudo netfilter-persistent save

# Verify
sudo iptables -t nat -L PREROUTING -n -v | grep $HOST_PORT
```

### 4.8 The Default Network XML — Annotated

```xml
<network>
  <name>default</name>              <!-- Network name, referenced in guest XML -->
  <uuid>...</uuid>                  <!-- Auto-generated UUID -->
  <forward mode='nat'>              <!-- Mode: nat | route | bridge | private | vepa | passthrough -->
    <nat>
      <port start='1024' end='65535'/>  <!-- NAT source port range -->
    </nat>
  </forward>
  <bridge name='virbr0'            <!-- Host bridge device name -->
          stp='on'                  <!-- Spanning Tree Protocol: on | off -->
          delay='0'/>               <!-- STP forward delay in seconds -->
  <mac address='52:54:00:...'/>    <!-- Bridge MAC — auto-generated -->
  <ip address='192.168.122.1'      <!-- Gateway IP (assigned to virbr0) -->
      netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' <!-- DHCP pool start -->
             end='192.168.122.254'/> <!-- DHCP pool end -->
      <!-- Static reservations go here: -->
      <!-- <host mac='52:54:00:...' name='vm1' ip='192.168.122.10'/> -->
    </dhcp>
  </ip>
</network>
```

---

## 5. Custom NAT / Isolated Networks

### 5.1 Create a Custom NAT Network

```bash
cat > /tmp/custom-nat.xml <<'EOF'
<network>
  <name>dev-nat</name>
  <forward mode='nat'/>
  <bridge name='virbr1' stp='on' delay='0'/>
  <ip address='10.10.10.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.10.10.10' end='10.10.10.200'/>
      <host mac='52:54:00:aa:bb:01' name='dev-vm01' ip='10.10.10.11'/>
      <host mac='52:54:00:aa:bb:02' name='dev-vm02' ip='10.10.10.12'/>
    </dhcp>
  </ip>
</network>
EOF

sudo virsh net-define /tmp/custom-nat.xml
sudo virsh net-start dev-nat
sudo virsh net-autostart dev-nat
virsh net-list --all
```

### 5.2 Modify an Existing Network (Live Update)

```bash
# Add a DHCP static reservation to a running network
virsh net-update dev-nat add ip-dhcp-host \
  "<host mac='52:54:00:cc:dd:ee' name='dev-vm03' ip='10.10.10.13'/>" \
  --live --config

# Add a DNS hostname entry
virsh net-update dev-nat add dns-host \
  "<host ip='10.10.10.13'><hostname>dev-vm03</hostname></host>" \
  --live --config

# Modify the DHCP range
virsh net-update dev-nat modify ip-dhcp-range \
  "<range start='10.10.10.50' end='10.10.10.250'/>" \
  --live --config

# List available update operations
virsh net-update dev-nat --help
```

### 5.3 Delete a Network

```bash
sudo virsh net-destroy dev-nat      # Stop (remove from host)
sudo virsh net-undefine dev-nat     # Remove definition (delete XML)
```

---

## 6. Bridged Networking (Ubuntu / Netplan)

Bridged networking connects guests directly to the physical LAN — as if they were physical machines. The host bridge (`br0`) ties together the physical NIC and the guest tap devices.

```
Guest VM (192.168.1.x from LAN DHCP)
         │
    tap device (vnet0)
         │
    br0 bridge ──────── enp3s0 (physical NIC, IP moved to br0)
         │
    Physical LAN switch
         │
    Router / DHCP server
```

### 6.1 Create a Bridge with Netplan (Ubuntu 20.04+)

**Step 1: Find your physical NIC name**

```bash
ip link show
# Identify the NIC connected to the LAN. Common names:
# enp3s0, ens3, enp0s3, eth0, eno1
# The one with the default route:
ip route get 8.8.8.8 | awk '/dev/{print $5}'

# Check current IP and config
ip addr show enp3s0
```

**Step 2: Write the Netplan bridge config**

```yaml
# /etc/netplan/01-kvm-bridge.yaml
# Replace enp3s0 with your actual NIC name
network:
  version: 2
  renderer: networkd          # Ubuntu server default; use NetworkManager for desktop
  ethernets:
    enp3s0:
      dhcp4: false            # Physical NIC gets no IP — bridge takes it over
      dhcp6: false
  bridges:
    br0:
      interfaces: [enp3s0]
      dhcp4: true             # Bridge gets the IP from LAN DHCP
      parameters:
        stp: false            # Disable STP for KVM — forward-delay causes VM timeouts
        forward-delay: 0
      mtu: 1500
```

**For a static IP on the bridge:**

```yaml
# /etc/netplan/01-kvm-bridge.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: false
  bridges:
    br0:
      interfaces: [enp3s0]
      dhcp4: false
      addresses: [192.168.1.100/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
        search: [example.com]
      parameters:
        stp: false
        forward-delay: 0
      mtu: 1500
```

**Step 3: Apply and verify**

```bash
# Validate without applying
sudo netplan try                     # Applies for 120s; auto-reverts if you don't confirm

# Apply permanently
sudo netplan apply

# Verify bridge
ip link show br0                     # State: UP
ip addr show br0                     # Should have the host's IP
brctl show br0                       # Should show enp3s0 as a slave interface
bridge link show                     # Alternative (iproute2)

# Test connectivity
ping -c3 8.8.8.8
```

**Step 4: Use the bridge in a VM**

```bash
# virt-install
virt-install \
  --name bridged-vm01 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20,bus=virtio \
  --cdrom /path/to/ubuntu.iso \
  --os-variant ubuntu22.04 \
  --network bridge=br0,model=virtio

# In guest XML (virsh edit <vmname>)
```

```xml
<interface type='bridge'>
  <source bridge='br0'/>
  <model type='virtio'/>
</interface>
```

### 6.2 Bridge with a Bonded NIC (HA / Redundancy)

```yaml
# /etc/netplan/01-kvm-bond-bridge.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: false
    enp4s0:
      dhcp4: false
  bonds:
    bond0:
      interfaces: [enp3s0, enp4s0]
      parameters:
        mode: active-backup       # or: balance-rr, 802.3ad (LACP), balance-alb
        primary: enp3s0
        mii-monitor-interval: 100
      dhcp4: false
  bridges:
    br0:
      interfaces: [bond0]
      dhcp4: true
      parameters:
        stp: false
        forward-delay: 0
```

### 6.3 Multiple Bridges (Multiple Physical NICs)

```yaml
# /etc/netplan/01-kvm-multi-bridge.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:       # Management / LAN NIC
      dhcp4: false
    enp4s0:       # Storage / data NIC
      dhcp4: false
  bridges:
    br0:          # LAN bridge — guests get LAN access
      interfaces: [enp3s0]
      dhcp4: true
      parameters:
        stp: false
        forward-delay: 0
    br1:          # Storage bridge — guests access storage network
      interfaces: [enp4s0]
      addresses: [10.0.1.1/24]
      parameters:
        stp: false
        forward-delay: 0
```

```bash
# Then in virt-install — attach guest to both bridges
virt-install ... \
  --network bridge=br0,model=virtio \   # LAN NIC (ens3 in guest)
  --network bridge=br1,model=virtio     # Storage NIC (ens7 in guest)
```

### 6.4 Verify Bridge Operation

```bash
# List all bridges and their enslaved interfaces
brctl show
# bridge name  bridge id          STP enabled  interfaces
# br0          8000.AABBCCDDEEFF  no           enp3s0
#                                              vnet0    ← Guest VM1 NIC
#                                              vnet1    ← Guest VM2 NIC

# Check bridge forwarding table (MAC table — like a switch's CAM table)
brctl showmacs br0

# Check STP state
brctl showstp br0

# Monitor bridge traffic in real time
sudo tcpdump -i br0 -n

# Count packets per second through the bridge
watch -n1 'ip -s link show br0'
```

---

## 7. Routed Networking

Routed networking gives guests a separate subnet that is *routed* (not NATed) to/from the LAN. Guests are reachable from the LAN if the upstream router knows how to route the guest subnet back.

```
Guest VM (10.20.30.x)
         │
    virbr2 (10.20.30.1) — route mode (no NAT)
         │
    Host IP routing table
         │
    Physical LAN (192.168.1.x)
         │
    Upstream router (must have static route: 10.20.30.0/24 via HOST_IP)
```

### 7.1 Define a Routed Network

```bash
cat > /tmp/routed-net.xml <<'EOF'
<network>
  <name>routed-net</name>
  <forward mode='route' dev='enp3s0'/>   <!-- Route via this host interface -->
  <bridge name='virbr2' stp='on' delay='0'/>
  <ip address='10.20.30.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.20.30.10' end='10.20.30.200'/>
    </dhcp>
  </ip>
</network>
EOF

sudo virsh net-define /tmp/routed-net.xml
sudo virsh net-start routed-net
sudo virsh net-autostart routed-net
```

### 7.2 Add Static Route on Upstream Router

The upstream router must know how to reach the guest subnet:

```bash
# Example: add a static route on another Ubuntu machine acting as router
sudo ip route add 10.20.30.0/24 via 192.168.1.100   # 192.168.1.100 = KVM host IP

# Persistent via Netplan on the router:
# routes:
#   - to: 10.20.30.0/24
#     via: 192.168.1.100
```

---

## 8. Isolated / Private Networking (No External Access)

Guests can communicate with each other and with the host but have no access to the physical network or the internet.

### 8.1 Define an Isolated Network

```bash
cat > /tmp/isolated-net.xml <<'EOF'
<network>
  <name>isolated</name>
  <!-- No <forward> element = isolated mode -->
  <bridge name='virbr3' stp='on' delay='0'/>
  <ip address='172.16.0.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='172.16.0.10' end='172.16.0.200'/>
    </dhcp>
  </ip>
</network>
EOF

sudo virsh net-define /tmp/isolated-net.xml
sudo virsh net-start isolated
sudo virsh net-autostart isolated
```

### 8.2 Use Case — Multi-VM Private Cluster

```bash
# Attach a VM to both the NAT network (internet) and the isolated network (internal)
virt-install \
  --name cluster-node01 \
  --memory 4096 --vcpus 2 \
  --disk size=20,bus=virtio \
  --import \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \     # outbound internet
  --network network=isolated,model=virtio      # internal cluster comms
```

---

## 9. VLAN Networking

VLANs allow multiple isolated network segments over a single physical NIC using 802.1Q tagging.

### 9.1 VLAN Overview for KVM

```
Physical NIC (enp3s0) — trunk port (carries tagged frames)
         │
    VLAN subinterfaces:
    enp3s0.10 (VLAN 10 — Production)
    enp3s0.20 (VLAN 20 — Dev/Test)
    enp3s0.30 (VLAN 30 — Management)
         │
    Bridge per VLAN:
    br-vlan10 → Production VMs
    br-vlan20 → Dev/Test VMs
    br-vlan30 → Management VMs
```

### 9.2 Configure VLANs via Netplan

```yaml
# /etc/netplan/01-kvm-vlan.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: false             # Trunk port — no IP on the raw NIC
  vlans:
    vlan10:
      id: 10
      link: enp3s0
      dhcp4: false
    vlan20:
      id: 20
      link: enp3s0
      dhcp4: false
    vlan30:
      id: 30
      link: enp3s0
      addresses: [192.168.30.1/24]   # Management VLAN: host has an IP here
  bridges:
    br-vlan10:
      interfaces: [vlan10]
      dhcp4: false
      parameters:
        stp: false
        forward-delay: 0
    br-vlan20:
      interfaces: [vlan20]
      dhcp4: false
      parameters:
        stp: false
        forward-delay: 0
    br-vlan30:
      interfaces: [vlan30]
      dhcp4: false
      parameters:
        stp: false
        forward-delay: 0
```

```bash
sudo netplan apply

# Verify VLAN interfaces
ip link show type vlan
# vlan10@enp3s0
# vlan20@enp3s0

# Verify bridges
brctl show
# br-vlan10 ... vlan10
# br-vlan20 ... vlan20
```

### 9.3 Connect a VM to a VLAN Bridge

```bash
# Production VM on VLAN 10
virt-install \
  --name prod-vm01 \
  --memory 4096 --vcpus 2 \
  --disk size=40,bus=virtio \
  --import \
  --os-variant ubuntu22.04 \
  --network bridge=br-vlan10,model=virtio

# Dev VM on VLAN 20
virt-install \
  --name dev-vm01 \
  --memory 2048 --vcpus 2 \
  --disk size=20,bus=virtio \
  --import \
  --os-variant ubuntu22.04 \
  --network bridge=br-vlan20,model=virtio
```

### 9.4 VLAN via libvirt Network Definition (portgroup)

libvirt supports VLAN portgroups natively in a network XML — useful for OVS environments:

```xml
<network>
  <name>vlan-net</name>
  <forward mode='bridge'/>
  <bridge name='ovs-br0'/>
  <virtualport type='openvswitch'/>
  <portgroup name='vlan10' default='no'>
    <vlan>
      <tag id='10'/>
    </vlan>
  </portgroup>
  <portgroup name='vlan20' default='no'>
    <vlan>
      <tag id='20'/>
    </vlan>
  </portgroup>
  <portgroup name='trunk' default='no'>
    <vlan trunk='yes'>
      <tag id='10'/>
      <tag id='20'/>
    </vlan>
  </portgroup>
</network>
```

---

## 10. Open vSwitch (OVS) Networking

Open vSwitch provides advanced software-defined networking: fine-grained flow control, tunnelling (VXLAN, GRE, Geneve), LACP, QoS, and OpenFlow support. Used in OpenStack, cloud, and multi-host VM environments.

### 10.1 Install and Start OVS

```bash
sudo apt install -y openvswitch-switch openvswitch-common
sudo systemctl enable --now openvswitch-switch
sudo systemctl status openvswitch-switch

# Verify OVS is working
sudo ovs-vsctl show
# version: ...
# ovs_version: "3.x.x"
```

### 10.2 Create an OVS Bridge

```bash
# Create the bridge
sudo ovs-vsctl add-br ovs-br0

# Add the physical NIC to the bridge (take the IP off the NIC first!)
sudo ip addr flush dev enp3s0
sudo ovs-vsctl add-port ovs-br0 enp3s0

# Assign the IP to the bridge instead
sudo ip addr add 192.168.1.100/24 dev ovs-br0
sudo ip link set ovs-br0 up
sudo ip route add default via 192.168.1.1

# Verify
sudo ovs-vsctl show
# Bridge ovs-br0
#     Port enp3s0
#         Interface enp3s0
#     Port ovs-br0
#         Interface ovs-br0 (internal)
```

### 10.3 Persist OVS Bridge via Netplan

```yaml
# /etc/netplan/01-ovs-bridge.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: false
  openvswitch:
    ports:
      - [enp3s0, ovs-br0]
  bridges:
    ovs-br0:
      openvswitch: {}
      interfaces: [enp3s0]
      addresses: [192.168.1.100/24]
      routes:
        - to: default
          via: 192.168.1.1
```

### 10.4 Define a libvirt Network Using OVS

```bash
cat > /tmp/ovs-network.xml <<'EOF'
<network>
  <name>ovs-net</name>
  <forward mode='bridge'/>
  <bridge name='ovs-br0'/>
  <virtualport type='openvswitch'/>
</network>
EOF

sudo virsh net-define /tmp/ovs-network.xml
sudo virsh net-start ovs-net
sudo virsh net-autostart ovs-net
```

### 10.5 Connect a VM to OVS

**In guest XML:**

```xml
<interface type='bridge'>
  <source bridge='ovs-br0'/>
  <virtualport type='openvswitch'>
    <parameters interfaceid='12345678-abcd-...'/>   <!-- Optional OVS port ID -->
  </virtualport>
  <model type='virtio'/>
</interface>
```

**Via virt-install:**

```bash
virt-install \
  --name ovs-vm01 \
  --memory 4096 --vcpus 2 \
  --disk size=20,bus=virtio \
  --import \
  --os-variant ubuntu22.04 \
  --network bridge=ovs-br0,virtualport_type=openvswitch,model=virtio
```

### 10.6 OVS VLAN Tagging per Port

```bash
# Assign a VLAN tag to a specific VM port (after the VM is running)
# Find the tap device name of the VM
TAPDEV=$(virsh domiflist ovs-vm01 | awk 'NR>2{print $1}')

# Tag the tap device on VLAN 10
sudo ovs-vsctl set port "$TAPDEV" tag=10

# Or make it a trunk port (carries multiple VLANs)
sudo ovs-vsctl set port "$TAPDEV" trunks=10,20,30

# Verify
sudo ovs-vsctl show
```

### 10.7 OVS VXLAN Tunnel (Multi-Host VM Networking)

```bash
# On Host 1 (192.168.1.10)
sudo ovs-vsctl add-port ovs-br0 vxlan0 -- \
  set interface vxlan0 type=vxlan \
  options:remote_ip=192.168.1.20 \
  options:key=100

# On Host 2 (192.168.1.20)
sudo ovs-vsctl add-port ovs-br0 vxlan0 -- \
  set interface vxlan0 type=vxlan \
  options:remote_ip=192.168.1.10 \
  options:key=100

# VMs on both hosts can now communicate as if on the same L2 network
# even though they're on different physical hosts

# Verify tunnel
sudo ovs-vsctl show
```

### 10.8 OVS QoS — Rate Limiting per VM

```bash
# Limit a VM's NIC to 100 Mbps ingress
TAPDEV=$(virsh domiflist <vmname> | awk 'NR>2{print $1}')

sudo ovs-vsctl set interface "$TAPDEV" \
  ingress_policing_rate=100000 \     # kbps
  ingress_policing_burst=10000

# Verify
sudo ovs-vsctl list interface "$TAPDEV" | grep -E 'ingress'
```

---

## 11. PCI Device Assignment (Passthrough)

PCI passthrough gives a VM exclusive ownership of a physical NIC. The guest sees and controls the hardware directly — no virtualisation overhead, best possible network performance.

### 11.1 Requirements

```bash
# 1. Check IOMMU support in CPU
grep -E 'vmx|svm' /proc/cpuinfo | head -2
dmesg | grep -i iommu
# Should show: "IOMMU enabled" or "Adding to iommu group"

# 2. Enable IOMMU in GRUB
sudo nano /etc/default/grub
# For Intel:
# GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt"
# For AMD:
# GRUB_CMDLINE_LINUX="amd_iommu=on iommu=pt"

sudo update-grub
sudo reboot

# 3. Verify IOMMU groups after reboot
find /sys/kernel/iommu_groups/ -type l | sort -V | head -20
```

### 11.2 Bind the NIC to vfio-pci

```bash
# Find the PCI address of the NIC to pass through
lspci | grep -i ethernet
# 0000:02:00.0 Ethernet controller: Intel Corporation 82574L Gigabit ...

PCI_ADDR="0000:02:00.0"

# Find current driver
lspci -k -s "$PCI_ADDR" | grep -i driver
# Kernel driver in use: e1000e  ← current driver

# Get vendor:device ID
lspci -n -s "$PCI_ADDR"
# 0000:02:00.0 0200: 8086:10d3   ← 8086:10d3

# Unbind from current driver
echo "$PCI_ADDR" | sudo tee /sys/bus/pci/devices/${PCI_ADDR}/driver/unbind

# Load vfio-pci module
sudo modprobe vfio-pci

# Bind to vfio-pci
echo "8086 10d3" | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id
# OR
echo "$PCI_ADDR" | sudo tee /sys/bus/pci/drivers/vfio-pci/bind

# Persist: add to /etc/modprobe.d/vfio.conf
echo "options vfio-pci ids=8086:10d3" | sudo tee /etc/modprobe.d/vfio.conf
echo "vfio-pci" | sudo tee /etc/modules-load.d/vfio.conf

# Verify
lspci -k -s "$PCI_ADDR" | grep -i driver
# Kernel driver in use: vfio-pci  ← correct
```

### 11.3 Attach the PCI Device to a Guest

```bash
# List all host PCI devices
virsh nodedev-list --cap pci | grep -i net

# Get detailed info for the device
virsh nodedev-dumpxml pci_0000_02_00_0

# Attach to a guest (VM must be shut off or support hotplug)
virsh attach-device <vmname> /tmp/pci-passthrough.xml --persistent
```

```xml
<!-- /tmp/pci-passthrough.xml -->
<hostdev mode='subsystem' type='pci' managed='yes'>
  <driver name='vfio'/>
  <source>
    <address domain='0x0000' bus='0x02' slot='0x00' function='0x0'/>
  </source>
</hostdev>
```

```bash
# Or specify at VM creation time (virt-install)
virt-install \
  --name pci-passthrough-vm \
  --memory 8192 --vcpus 4 \
  --disk size=50,bus=virtio \
  --import \
  --os-variant ubuntu22.04 \
  --host-device 0000:02:00.0
```

---

## 12. SR-IOV (PCIe Virtual Functions)

SR-IOV (Single Root I/O Virtualization) splits one physical NIC into multiple Virtual Functions (VFs). Each VF can be assigned to a guest — high performance like PCI passthrough, but scalable (up to 256 VFs per physical NIC).

### 12.1 Requirements and Verification

```bash
# Check if NIC supports SR-IOV
lspci | grep -i ethernet
ethtool -i enp3s0 | grep driver
# Must support VFs — check NIC vendor docs

# Check IOMMU is enabled (same as PCI passthrough)
dmesg | grep -i iommu

# Check current VF count
cat /sys/class/net/enp3s0/device/sriov_numvfs   # Current: 0
cat /sys/class/net/enp3s0/device/sriov_totalvfs  # Max supported
```

### 12.2 Create Virtual Functions

```bash
# Enable 4 VFs on enp3s0
echo 4 | sudo tee /sys/class/net/enp3s0/device/sriov_numvfs

# Verify VFs created
ip link show enp3s0
# Shows: vf 0, vf 1, vf 2, vf 3

lspci | grep "Virtual Function"
# 0000:02:10.0 ... SR-IOV: Virtual Function
# 0000:02:10.2 ...
# 0000:02:10.4 ...
# 0000:02:10.6 ...

# Persist across reboots (create a udev rule or use a service)
cat > /etc/rc.local <<'EOF'
#!/bin/bash
echo 4 > /sys/class/net/enp3s0/device/sriov_numvfs
exit 0
EOF
sudo chmod +x /etc/rc.local
```

### 12.3 Bind VFs to vfio-pci and Assign to Guests

```bash
# Find VF PCI addresses
lspci | grep "Virtual Function"
# 0000:02:10.0 — VF 0
# 0000:02:10.2 — VF 1

# Bind VF 0 to vfio-pci
VF_PCI="0000:02:10.0"
echo "$VF_PCI" | sudo tee /sys/bus/pci/devices/${VF_PCI}/driver/unbind
echo "$VF_PCI" | sudo tee /sys/bus/pci/drivers/vfio-pci/bind

# Assign to a VM
virsh attach-device <vmname> /tmp/vf-device.xml --persistent
```

```xml
<!-- /tmp/vf-device.xml -->
<hostdev mode='subsystem' type='pci' managed='yes'>
  <driver name='vfio'/>
  <source>
    <address domain='0x0000' bus='0x02' slot='0x10' function='0x0'/>
  </source>
</hostdev>
```

---

## 13. vhost-net — Kernel-Level Virtio Backend

### 13.1 What Is vhost-net?

```
WITHOUT vhost-net (userspace, slower):
Guest → virtio ring → QEMU process (user space) → kernel network stack → physical NIC

WITH vhost-net (kernel-level, faster):
Guest → virtio ring → vhost-net kernel module → kernel network stack → physical NIC
                       ↑ Bypasses QEMU entirely for packet I/O
```

vhost-net moves virtio packet processing from QEMU (user space) into the kernel, reducing context switches and CPU overhead — especially beneficial for high-throughput workloads.

### 13.2 Verify vhost-net is Loaded

```bash
lsmod | grep vhost
# vhost_net   ...   0
# vhost       ...   1 vhost_net

# Load if not present
sudo modprobe vhost_net

# Persist
echo "vhost_net" | sudo tee /etc/modules-load.d/vhost-net.conf

# Verify it's in use for running VMs
ls /dev/vhost-net   # Must exist
```

vhost-net is **enabled by default** for all virtio interfaces when the module is loaded. No explicit configuration is needed to enable it.

### 13.3 Disable vhost-net for a Specific VM NIC

Disable when: host→guest UDP traffic causes packet drops because the guest processes incoming data slower than the host sends it. vhost-net's in-kernel socket buffer fills up faster → more drops. Disabling forces QEMU to handle packets, which naturally slows the send rate.

```xml
<!-- In guest XML (virsh edit <vmname>) -->
<!-- Setting driver name="qemu" forces userspace processing = disables vhost-net -->
<interface type='network'>
  <source network='default'/>
  <model type='virtio'/>
  <driver name='qemu'/>        <!-- ← This disables vhost-net -->
</interface>
```

```bash
# Apply without VM reboot (hot update)
virsh update-device <vmname> /tmp/interface-no-vhost.xml --live
```

### 13.4 Enable vhost-net Zero-Copy Transmit

Zero-copy transmit reduces CPU usage for large transmit packets by avoiding a memory copy step. Disabled by default on Ubuntu (considered experimental).

```bash
# Check current state
cat /sys/module/vhost_net/parameters/experimental_zcopytx
# 0 = disabled (default)

# Enable permanently (add kernel module parameter)
echo "options vhost_net experimental_zcopytx=1" | \
  sudo tee /etc/modprobe.d/vhost-net-zcopy.conf

# Reload the module to apply
sudo modprobe -r vhost_net
sudo modprobe vhost_net

# Verify
cat /sys/module/vhost_net/parameters/experimental_zcopytx
# 1 = enabled

# Disable again
sudo modprobe -r vhost_net
sudo modprobe vhost_net experimental_zcopytx=0
cat /sys/module/vhost_net/parameters/experimental_zcopytx
# 0

# Non-permanent enable (lost on reboot)
echo 1 | sudo tee /sys/module/vhost_net/parameters/experimental_zcopytx
```

> **When to use zero-copy:** High-throughput bulk transfer workloads (file servers, backup agents, data pipeline VMs). Avoid for workloads with many small packets (latency-sensitive, OLTP).

### 13.5 vhost-net vs vhost-user (DPDK)

| Feature | vhost-net | vhost-user (DPDK) |
|---|---|---|
| Location | Linux kernel | User space (DPDK) |
| Performance | Good | Excellent |
| CPU model | Interrupt driven | Polling (dedicated cores) |
| Latency | Low | Ultra-low |
| Setup complexity | None (automatic) | High (DPDK config) |
| Use case | General VM networking | NFV, telco, HPC |

---

## 14. Macvtap / macvlan Networking

Macvtap creates a virtual NIC directly on a physical NIC — no bridge needed. The guest gets a MAC address on the physical network without a bridge device on the host.

### 14.1 How Macvtap Works

```
Physical NIC (enp3s0) — has a macvtap attached directly
         │
    macvtap0 → Guest VM (gets its own MAC, LAN-visible IP)

NOTE: The HOST cannot communicate with the guest via macvtap
      (host→guest traffic goes to the physical network, not back via macvtap)
      Use a separate bridge NIC or additional NAT NIC for host↔guest communication.
```

### 14.2 Macvtap Modes

| Mode | Description | Use case |
|---|---|---|
| `vepa` | VEPA — VM-to-VM traffic goes via external switch | Enterprise managed switches |
| `bridge` | VMs on same host can communicate directly | Most common |
| `private` | VMs cannot communicate with each other | Strict isolation |
| `passthrough` | Exclusive access to the physical NIC | High performance |

### 14.3 Configure Macvtap in Guest XML

```xml
<!-- bridge mode — VMs on same host can talk to each other -->
<interface type='direct'>
  <source dev='enp3s0' mode='bridge'/>
  <model type='virtio'/>
</interface>

<!-- vepa mode — traffic goes through physical switch -->
<interface type='direct'>
  <source dev='enp3s0' mode='vepa'/>
  <model type='virtio'/>
</interface>
```

```bash
# Via virt-install
virt-install \
  --name macvtap-vm01 \
  --memory 2048 --vcpus 2 \
  --disk size=20,bus=virtio \
  --import \
  --os-variant ubuntu22.04 \
  --network type=direct,source=enp3s0,source_mode=bridge,model=virtio
```

---

## 15. Guest XML Network Interface Configuration

### 15.1 Full Interface XML Reference

```xml
<interface type='network'>           <!-- type: network | bridge | direct | hostdev | ethernet -->

  <!-- Source — depends on type -->
  <source network='default'/>        <!-- For type='network' -->
  <!-- <source bridge='br0'/>            For type='bridge' -->
  <!-- <source dev='enp3s0' mode='bridge'/>   For type='direct' (macvtap) -->

  <!-- Identity -->
  <mac address='52:54:00:aa:bb:cc'/>  <!-- Optional; auto-generated if omitted -->
  <target dev='vnet0'/>               <!-- tap device name on host (auto if omitted) -->
  <alias name='net0'/>                <!-- Guest-visible device alias -->

  <!-- Performance model -->
  <model type='virtio'/>             <!-- virtio | e1000 | rtl8139 | vmxnet3 -->

  <!-- Driver / backend -->
  <driver name='vhost'              <!-- vhost (kernel) | qemu (userspace) -->
          queues='4'                <!-- Multi-queue: match vCPU count for best perf -->
          rx_queue_size='256'       <!-- Receive ring size (power of 2) -->
          tx_queue_size='256'/>     <!-- Transmit ring size -->

  <!-- Bandwidth / QoS throttling -->
  <bandwidth>
    <inbound average='10240'        <!-- kbps average -->
              peak='20480'          <!-- kbps burst peak -->
              burst='1024'/>        <!-- KB burst bucket size -->
    <outbound average='10240'
               peak='20480'
               burst='1024'/>
  </bandwidth>

  <!-- VLAN tagging (for OVS portgroup networks) -->
  <vlan>
    <tag id='10'/>
  </vlan>

  <!-- VirtualPort (for OVS) -->
  <virtualport type='openvswitch'>
    <parameters interfaceid='...'/>
  </virtualport>

  <!-- Link state -->
  <link state='up'/>               <!-- up | down — can be changed without restart -->

  <!-- ROM boot (for PXE) -->
  <rom enabled='yes' bar='on'/>

</interface>
```

### 15.2 NIC Model Comparison

| Model | Guest driver | Performance | Compatibility | Use case |
|---|---|---|---|---|
| `virtio` | `virtio_net` | Best | Linux 2.6.24+ / Windows (with drivers) | All Ubuntu guests |
| `e1000` | `e1000` | Medium | Universally supported | Windows/legacy guests |
| `rtl8139` | `8139too` | Lowest | Widest compatibility | Very old OSes |
| `vmxnet3` | VMware driver | Good | VMware-specific | VMware guest compat |

### 15.3 Multi-Queue Virtio (Network Performance)

Multi-queue allows the guest to use multiple CPU cores for network processing.

```xml
<interface type='network'>
  <source network='default'/>
  <model type='virtio'/>
  <driver name='vhost' queues='4'/>   <!-- Set to number of vCPUs (max 8 recommended) -->
</interface>
```

Inside the guest, enable the queues:

```bash
# Inside the guest
sudo ethtool -L ens3 combined 4    # Set 4 combined queue channels

# Verify
ethtool -l ens3
```

### 15.4 Live Hotplug / Hotunplug of NICs

```bash
# Attach a new NIC to a running VM
cat > /tmp/new-nic.xml <<'EOF'
<interface type='network'>
  <source network='default'/>
  <model type='virtio'/>
</interface>
EOF

virsh attach-device <vmname> /tmp/new-nic.xml --live --persistent

# Detach a NIC from a running VM (find MAC first)
virsh domiflist <vmname>
virsh detach-interface <vmname> network --mac 52:54:00:xx:xx:xx --live --persistent
```

---

## 16. UFW Firewall and libvirt

libvirt manages its own iptables/nftables rules. Ubuntu's UFW can interfere if not configured correctly.

### 16.1 Why UFW Breaks VM Networking

UFW's default FORWARD policy is `DROP`. libvirt requires forwarding between guest tap devices and the physical NIC. Without correct UFW rules, guest VMs lose network connectivity when UFW is enabled.

### 16.2 Required UFW Configuration

```bash
# Step 1: Allow forwarding in UFW
sudo nano /etc/default/ufw
# Change: DEFAULT_FORWARD_POLICY="DROP"
# To:     DEFAULT_FORWARD_POLICY="ACCEPT"

# Step 2: Enable IP forwarding in UFW's sysctl
sudo nano /etc/ufw/sysctl.conf
# Uncomment or add:
# net/ipv4/ip_forward=1
# net/ipv6/conf/default/forwarding=1

# Step 3: Allow traffic on libvirt bridges
sudo ufw allow in on virbr0 to any
sudo ufw allow out on virbr0 to any
sudo ufw allow in on br0 to any
sudo ufw allow out on br0 to any

# Step 4: Allow traffic on additional custom bridges
for BRIDGE in virbr1 virbr2 br-vlan10 br-vlan20; do
  sudo ufw allow in on $BRIDGE to any
  sudo ufw allow out on $BRIDGE to any
done

# Step 5: Reload UFW
sudo ufw reload
sudo ufw status verbose

# Verify forwarding is working
sudo iptables -L FORWARD -n -v | head -20
```

### 16.3 Preserve libvirt's iptables Rules After UFW Operations

libvirt adds iptables rules at daemon start. If UFW flushes all rules, the VM network breaks.

```bash
# Tell libvirt to re-add its iptables rules
sudo systemctl restart libvirtd

# Or use libvirt's own firewall driver
# In /etc/libvirt/network.conf:
# firewall_backend = "iptables"   # or "nftables"
```

### 16.4 Ubuntu 22.04 — nftables Backend

Ubuntu 22.04+ uses nftables by default. libvirt can use either backend:

```bash
# Check which backend libvirt is using
sudo virsh --version
cat /etc/libvirt/network.conf | grep firewall_backend

# Force iptables backend (more widely tested)
sudo nano /etc/libvirt/network.conf
# firewall_backend = "iptables"
sudo systemctl restart libvirtd

# Or use nftables backend
# firewall_backend = "nftables"

# Inspect nftables rules libvirt creates
sudo nft list ruleset | grep -A5 libvirt
```

---

## 17. IP Forwarding and Routing

### 17.1 Enable IP Forwarding (Permanent)

```bash
# Create a dedicated sysctl config
sudo tee /etc/sysctl.d/99-kvm-network.conf <<'EOF'
# KVM networking requirements
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# Prevent RP filter from blocking VM traffic
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0

# Increase conntrack table (for many simultaneous VM connections)
net.netfilter.nf_conntrack_max = 131072

# Bridge filtering (required for libvirt bridged networking)
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl -p /etc/sysctl.d/99-kvm-network.conf

# Verify
sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 1
```

### 17.2 Load Bridge Netfilter Module

```bash
# Required for bridge-nf-call-iptables to work
sudo modprobe br_netfilter

# Persist
echo "br_netfilter" | sudo tee /etc/modules-load.d/br_netfilter.conf
```

### 17.3 Manual NAT Rule (Without libvirt's Default Network)

```bash
# If you need to NAT a custom bridge manually
BRIDGE="br-custom"
BRIDGE_SUBNET="10.30.0.0/24"
PHYSICAL_IF="enp3s0"

sudo iptables -t nat -A POSTROUTING \
  -s "$BRIDGE_SUBNET" \
  ! -d "$BRIDGE_SUBNET" \
  -j MASQUERADE

sudo iptables -A FORWARD \
  -i "$BRIDGE" -o "$PHYSICAL_IF" \
  -j ACCEPT

sudo iptables -A FORWARD \
  -i "$PHYSICAL_IF" -o "$BRIDGE" \
  -m state --state RELATED,ESTABLISHED \
  -j ACCEPT

# Save rules
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

---

## 18. DNS and DHCP Inside Virtual Networks

libvirt runs a `dnsmasq` instance per virtual network for DHCP and DNS.

### 18.1 dnsmasq Configuration Files

```bash
# Config for the default network
cat /var/lib/libvirt/dnsmasq/default.conf

# Current DHCP leases
cat /var/lib/libvirt/dnsmasq/default.leases

# Hosts file (static entries)
cat /var/lib/libvirt/dnsmasq/default.addnhosts
```

### 18.2 Add DNS Records to a Virtual Network

```bash
# Add a DNS A record (hostname → IP)
virsh net-update default add dns-host \
  "<host ip='192.168.122.50'>
     <hostname>myapp.internal</hostname>
   </host>" \
  --live --config

# Add a DNS forwarder (for a specific domain)
virsh net-update default add dns-forwarder \
  "<forwarder domain='corp.example.com' addr='10.0.0.1'/>" \
  --live --config

# Add a DNS TXT record
virsh net-update default add dns-txt \
  "<txt name='_test.default' value='hello'/>" \
  --live --config

# Remove a DNS host entry
virsh net-update default delete dns-host \
  "<host ip='192.168.122.50'/>" \
  --live --config
```

### 18.3 Custom dnsmasq Options

For options not exposed via virsh net-update, add a `dnsmasq` options file:

```bash
# Create a custom options file for the default network
cat > /etc/libvirt/hooks/network.d/default-dnsmasq-options <<'EOF'
# Additional dnsmasq options for the default network
# These are passed to the dnsmasq process
domain=vm.internal
local=/vm.internal/
expand-hosts
EOF

# Or edit the network XML directly (virsh edit-network equivalent)
NETXML=$(virsh net-dumpxml default)
# Add <dns> section to the network XML
```

```xml
<!-- DNS section inside network XML -->
<network>
  <name>default</name>
  ...
  <dns>
    <forwarder addr='8.8.8.8'/>
    <forwarder domain='corp.internal' addr='10.0.0.1'/>
    <txt name='example' value='test'/>
    <host ip='192.168.122.10'>
      <hostname>db01</hostname>
      <hostname>db01.vm.internal</hostname>
    </host>
  </dns>
  ...
</network>
```

### 18.4 Static DHCP Reservations

```bash
# Reserve an IP for a specific MAC (always gets the same IP)
virsh net-update default add ip-dhcp-host \
  "<host mac='52:54:00:aa:bb:cc' name='web01' ip='192.168.122.50'/>" \
  --live --config

# List all static reservations
virsh net-dumpxml default | grep -A2 '<host'

# Remove a reservation
virsh net-update default delete ip-dhcp-host \
  "<host mac='52:54:00:aa:bb:cc'/>" \
  --live --config
```

---

## 19. Network Performance Tuning

### 19.1 Always Use virtio NIC (Baseline Rule)

```bash
# Verify guest is using virtio
virsh domiflist <vmname>
# Interface   Type     Source    Model    MAC
# vnet0       network  default   virtio   ...  ← correct

# Inside guest — verify driver
lspci | grep -i ethernet
ethtool -i ens3 | grep driver
# driver: virtio_net  ← correct
```

### 19.2 Enable Multi-Queue virtio

```xml
<!-- In guest XML — set queues to match vCPU count -->
<interface type='network'>
  <source network='default'/>
  <model type='virtio'/>
  <driver name='vhost' queues='4'/>
</interface>
```

```bash
# Inside guest — activate all queues
sudo ethtool -L ens3 combined 4
# Verify
ethtool -l ens3
# Current hardware settings for ens3:
#   Combined: 4
```

### 19.3 Increase Virtio Ring Buffer Sizes

Larger ring buffers reduce packet drops under burst traffic:

```xml
<interface type='network'>
  <source network='default'/>
  <model type='virtio'/>
  <driver name='vhost'
          queues='4'
          rx_queue_size='1024'
          tx_queue_size='1024'/>
</interface>
```

```bash
# Also increase inside guest via ethtool
sudo ethtool -G ens3 rx 1024 tx 1024
# Verify
ethtool -g ens3
```

### 19.4 Disable TSO/GSO Offloading (If Causing Issues)

In some cases, offloading causes interoperability issues (especially with tunnels):

```bash
# Inside guest — check offload features
ethtool -k ens3 | grep -E 'tcp|generic|scatter'

# Disable if needed (non-persistent — for testing only)
sudo ethtool -K ens3 tso off gso off gro off

# Persistent (add to /etc/network/interfaces or Netplan post-up hook)
sudo ethtool -K ens3 tso on gso on gro on    # Re-enable
```

### 19.5 Use vhost-net Zero-Copy for High Throughput

```bash
# Check current state
cat /sys/module/vhost_net/parameters/experimental_zcopytx

# Enable for bulk transfer workloads
echo "options vhost_net experimental_zcopytx=1" | \
  sudo tee /etc/modprobe.d/vhost-net-zcopy.conf
sudo modprobe -r vhost_net && sudo modprobe vhost_net
```

### 19.6 Tune QEMU Network Buffer Sizes

```xml
<!-- Tune socket buffer size for the tap device -->
<interface type='network'>
  <source network='default'/>
  <model type='virtio'/>
  <tune>
    <sndbuf>1048576</sndbuf>   <!-- 1MB send buffer — larger = fewer drops -->
  </tune>
</interface>
```

### 19.7 Bandwidth QoS Throttling per VM

```xml
<!-- Limit this VM's NIC to 100 Mbps in and out -->
<interface type='network'>
  <source network='default'/>
  <model type='virtio'/>
  <bandwidth>
    <inbound  average='102400' peak='204800' burst='1024'/>   <!-- kbps -->
    <outbound average='102400' peak='204800' burst='1024'/>
  </bandwidth>
</interface>
```

```bash
# Apply live without rebooting
virsh update-device <vmname> /tmp/throttled-nic.xml --live
```

### 19.8 Performance Benchmark Inside Guest

```bash
# Inside two VMs on the same host — test VM-to-VM bandwidth
# On VM2 (receiver): install iperf3
sudo apt install iperf3
iperf3 -s -p 5201

# On VM1 (sender):
sudo apt install iperf3
GUEST2_IP="192.168.122.101"
iperf3 -c $GUEST2_IP -p 5201 -t 30 -P 4   # 4 parallel streams, 30 seconds

# Expected: ~10 Gbps between VMs on the same host with virtio
```

---

## 20. Monitoring and Diagnostics

### 20.1 Per-VM Network Statistics

```bash
# Per-interface statistics for a running VM
virsh domifstat <vmname> vnet0
# ens3 rx_bytes    rx_packets    rx_errs    rx_drop
# ens3 tx_bytes    tx_packets    tx_errs    tx_drop

# All stats via domstats
virsh domstats <vmname> --interface

# Historical I/O monitoring with dstat
sudo apt install dstat
dstat -n --net-packets
```

### 20.2 Monitor All Bridge Traffic

```bash
# Capture all traffic on the libvirt NAT bridge
sudo tcpdump -i virbr0 -n -v

# Capture only DHCP
sudo tcpdump -i virbr0 -n port 67 or port 68

# Capture traffic to/from a specific guest IP
sudo tcpdump -i virbr0 -n host 192.168.122.50

# Capture on a physical bridge (bridged networking)
sudo tcpdump -i br0 -n

# Write capture to file for analysis in Wireshark
sudo tcpdump -i virbr0 -n -w /tmp/kvm-capture.pcap
```

### 20.3 Inspect tap Devices

```bash
# List all tap devices and their bridges
ip link show type tun | grep -A1 vnet

# Find which VM owns a tap device
# (Map vnet0 → virsh domain)
for VM in $(virsh list --name); do
  TAPS=$(virsh domiflist $VM | awk 'NR>2{print $1}')
  for TAP in $TAPS; do
    echo "${VM}: ${TAP}"
  done
done

# Check tap device statistics
ip -s link show vnet0
```

### 20.4 Check iptables Rules libvirt Created

```bash
# NAT rules
sudo iptables -t nat -L -n -v | grep -E 'virbr|MASQUERADE'

# Forward rules
sudo iptables -L FORWARD -n -v | grep virbr

# All rules in table format
sudo iptables -L -n -v -t nat
sudo iptables -L -n -v -t filter

# Ubuntu 22.04: use nftables
sudo nft list ruleset
```

### 20.5 Find Guest IP Addresses

```bash
# Method 1: DHCP lease table (NAT networks only)
virsh net-dhcp-leases default

# Method 2: Guest agent (most reliable — requires qemu-guest-agent in guest)
virsh domifaddr <vmname> --source agent

# Method 3: ARP table (works briefly after a guest pings something)
arp -n | grep $(virsh domiflist <vmname> | awk 'NR>2{print $5}')

# Method 4: Scan the subnet
sudo nmap -sn 192.168.122.0/24

# Method 5: Read from guest's tap device MAC
MAC=$(virsh domiflist <vmname> | awk 'NR>2{print $5}')
virsh net-dhcp-leases default | grep "$MAC"
```

### 20.6 Real-Time Network Traffic per VM

```bash
# Install iftop for per-interface monitoring
sudo apt install iftop

# Monitor the bridge (shows all VM traffic)
sudo iftop -i virbr0

# Monitor a specific tap device (single VM)
TAP=$(virsh domiflist <vmname> | awk 'NR>2{print $1}')
sudo iftop -i "$TAP"

# Or use bmon for bandwidth monitoring
sudo apt install bmon
bmon -p virbr0
```

---

## 21. Troubleshooting Quick Reference

### Guest Has No Network Connectivity

```bash
# Step 1: Check the virtual network is running
virsh net-list --all
virsh net-start default

# Step 2: Check IP forwarding is on
sysctl net.ipv4.ip_forward    # Must be 1
sudo sysctl -w net.ipv4.ip_forward=1   # Fix temporarily

# Step 3: Check libvirt's iptables rules exist
sudo iptables -L FORWARD -n -v | grep virbr
# If empty — libvirt rules were flushed:
sudo systemctl restart libvirtd

# Step 4: Check UFW is not blocking forwarding
sudo ufw status
sudo iptables -L FORWARD -n | head -5
# If policy is DROP: sudo ufw default allow FORWARD (edit /etc/default/ufw)

# Step 5: Check tap device is enslaved to bridge
brctl show virbr0     # vnet0 should appear under virbr0

# Step 6: Check inside the guest
virsh console <vmname>
ip link show           # ens3 must be UP
ip addr show ens3      # Must have an IP
ip route               # Must have a default route
ping 192.168.122.1     # Ping the gateway (virbr0)
```

### Guest Gets Wrong IP / Same IP as Another VM

```bash
# Cause: Cloned VM has same machine-id → same DHCP client ID → same IP lease

# Fix (inside the guest)
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
sudo reboot
# After reboot: new machine-id → different DHCP lease
```

### Bridge Not Forwarding Packets

```bash
# Check br_netfilter is loaded
lsmod | grep br_netfilter

# Load it
sudo modprobe br_netfilter

# Check bridge-nf sysctl
sysctl net.bridge.bridge-nf-call-iptables    # Must be 1
sudo sysctl -w net.bridge.bridge-nf-call-iptables=1

# Check STP is not adding delay (disable for VM bridges)
brctl showstp br0 | grep state
# Should NOT show "listening" or "learning" — only "forwarding"
# Fix: sudo brctl stp br0 off (or set forward-delay=0 in Netplan)
```

### PXE Boot Fails (Guest Can't Get IP via PXE)

```bash
# Verify the VM NIC is on the bridge (not NAT)
virsh domiflist <vmname>
# Source must be br0 (bridged), not virbr0 (NAT)

# Check DHCP packets are reaching the bridge
sudo tcpdump -i br0 -n port 67 or port 68

# Check firewall isn't blocking DHCP
sudo ufw allow in on br0 to any port 67 proto udp
sudo ufw allow in on br0 to any port 68 proto udp
```

### Netplan Bridge Apply Fails

```bash
# Test Netplan config for syntax errors
sudo netplan --debug generate

# Apply with auto-revert (120s to confirm)
sudo netplan try

# Check journal for errors
journalctl -u systemd-networkd --since "5 min ago"

# Common issue: renderer mismatch (desktop uses NetworkManager, not networkd)
# Check which renderer is active
networkctl status
systemctl is-active systemd-networkd
systemctl is-active NetworkManager
```

### vhost-net Causing UDP Packet Loss

```bash
# Symptom: guest processes UDP slower than host sends → buffer overflow → drops

# Diagnosis
ss -u -p    # Check UDP socket receive buffer usage inside guest
netstat -suno | grep -i "receive buffer"

# Fix: disable vhost-net for that NIC in guest XML
virsh edit <vmname>
# Change <driver name='vhost'/> to <driver name='qemu'/>

# Or increase UDP socket buffer size (inside guest)
sudo sysctl -w net.core.rmem_max=16777216
sudo sysctl -w net.core.rmem_default=16777216
echo "net.core.rmem_max=16777216" | sudo tee -a /etc/sysctl.d/99-udp-buffers.conf
```

### libvirt Network Does Not Survive Reboot

```bash
# Ensure autostart is set
virsh net-autostart default
virsh net-autostart custom-net

# Verify
virsh net-info default | grep Autostart
# Autostart:      yes

# Also check libvirtd itself autostarts
sudo systemctl is-enabled libvirtd
# enabled  ← correct
# If not: sudo systemctl enable libvirtd

# Ubuntu 22.04+:
sudo systemctl enable --now virtqemud
sudo systemctl enable --now virtnetworkd
```

### OVS Bridge Not Persisting After Reboot

```bash
# Check if openvswitch-switch service is enabled
sudo systemctl is-enabled openvswitch-switch
sudo systemctl enable openvswitch-switch

# OVS stores its config in a database that persists automatically
# The bridge should survive reboots once created
sudo ovs-vsctl show

# If using Netplan for OVS, verify the YAML file is correct
sudo netplan --debug generate
sudo netplan apply
```

---

## 22. Quick-Reference Cheat Sheet

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 VIRTUAL NETWORK MANAGEMENT (virsh net-*)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 virsh net-list --all                    List all networks
 virsh net-info <net>                    Network details
 virsh net-dumpxml <net>                 Full XML definition
 virsh net-define /path/net.xml         Define network from XML
 virsh net-start <net>                  Start network
 virsh net-destroy <net>                Stop network
 virsh net-autostart <net>              Auto-start on host boot
 virsh net-autostart <net> --disable    Disable autostart
 virsh net-undefine <net>              Delete network definition
 virsh net-edit <net>                   Edit network XML
 virsh net-dhcp-leases <net>           Show DHCP leases

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 LIVE NETWORK UPDATES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Add DHCP host:
   virsh net-update <net> add ip-dhcp-host \
     "<host mac='52:54:00:xx:xx:xx' name='vm1' ip='x.x.x.x'/>" \
     --live --config

 Add DNS host:
   virsh net-update <net> add dns-host \
     "<host ip='x.x.x.x'><hostname>vm1.internal</hostname></host>" \
     --live --config

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 NETWORK TYPES — virt-install FLAGS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 NAT              --network network=default,model=virtio
 Bridge           --network bridge=br0,model=virtio
 Isolated         --network network=isolated,model=virtio
 Macvtap          --network type=direct,source=enp3s0,source_mode=bridge
 OVS              --network bridge=ovs-br0,virtualport_type=openvswitch
 No NIC           --network=none
 Multi-NIC        --network ... --network ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 BRIDGE MANAGEMENT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 brctl show                             List bridges and ports
 brctl showmacs <bridge>               MAC forwarding table
 brctl showstp <bridge>                STP state
 ip link show type bridge              iproute2 bridge list
 bridge link show                      Bridge ports (iproute2)
 sudo tcpdump -i <bridge> -n           Capture bridge traffic

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 OVS MANAGEMENT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 sudo ovs-vsctl show                   Overview of OVS config
 sudo ovs-vsctl add-br <bridge>        Create OVS bridge
 sudo ovs-vsctl add-port <br> <port>  Add port to bridge
 sudo ovs-vsctl del-br <bridge>        Delete bridge
 sudo ovs-vsctl set port <p> tag=<id> Assign VLAN tag
 sudo ovs-ofctl dump-flows <bridge>    Show OpenFlow rules
 sudo ovs-dpctl show                   Datapath overview

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 vhost-net
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 lsmod | grep vhost                    Is vhost-net loaded?
 sudo modprobe vhost_net               Load module
 Disable per-NIC:    <driver name='qemu'/> in XML
 Enable zero-copy:   echo "options vhost_net experimental_zcopytx=1" \
                     > /etc/modprobe.d/vhost-net-zcopy.conf
 Verify zero-copy:   cat /sys/module/vhost_net/parameters/experimental_zcopytx

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 DIAGNOSTICS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 virsh domiflist <vm>                  VM's NIC list + tap device
 virsh domifstat <vm> <tap>           NIC packet/byte counters
 virsh domifaddr <vm> --source agent  Guest IPs (via agent)
 virsh net-dhcp-leases default        DHCP lease table
 sudo iptables -t nat -L -n -v        NAT rules (libvirt-added)
 sudo iptables -L FORWARD -n -v       Forward rules
 sudo tcpdump -i virbr0 -n            Capture NAT bridge traffic
 sudo tcpdump -i br0 -n               Capture bridge traffic
 sysctl net.ipv4.ip_forward           Must be 1

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 KEY UBUNTU vs RHEL DIFFERENCES (networking context)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Network config      Netplan YAML (/etc/netplan/)
                     (not ifcfg-* scripts in /etc/sysconfig/)
 Firewall            UFW / nftables  (not firewalld)
 nftables backend    Default on Ubuntu 22.04+
 Netplan apply       sudo netplan apply
 Netplan test        sudo netplan try  (auto-reverts in 120s)
 Default NIC name    enp3s0 / ens3  (not eth0)
 NIC in VM           ens3 (virtio) — confirm with: ip link show
 IP forward file     /etc/sysctl.d/99-*.conf  (not /etc/sysctl.conf)
 OVS package         openvswitch-switch  (same as RHEL)
 VLAN package        vlan  (also: ip link add vlan type vlan)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

> **References:**
> - Ubuntu Server Guide — Networking: https://ubuntu.com/server/docs/network-configuration
> - Ubuntu Server Guide — Virtualisation: https://ubuntu.com/server/docs/virtualization-libvirt
> - libvirt Network XML format: https://libvirt.org/formatnetwork.html
> - libvirt Networking: https://wiki.libvirt.org/page/Networking
> - Open vSwitch docs: https://docs.openvswitch.org/
> - Netplan reference: https://netplan.io/reference/
> - `man virsh` · `man brctl` · `man ovs-vsctl` · `man netplan`
