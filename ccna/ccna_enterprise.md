# 🌐 CCNA Enterprise (200-301) — 30-Day Study Plan

> **Goal:** Become a network expert, not just pass the exam.
> **Daily commitment:** ~3–4 hours (1.5h theory + 1.5–2h hands-on + 30min notes to GitHub)
> **Structure:** Theory → Lab (Packet Tracer / CLI) → Document to GitHub

---

## 🗂️ GitHub Second Brain Setup (Do This on Day 0)

Create a GitHub repo called `ccna-enterprise-notes`. Use this structure:

```
ccna-enterprise-notes/
├── README.md                        # Master index + progress tracker
├── QUICK-REFERENCE.md               # Your personal cheat sheet (update daily)
├── _template.md                     # Note template (copy for each topic)
├── 01-network-fundamentals/
│   ├── osi-tcp-ip-models.md
│   ├── ipv4-subnetting.md
│   ├── ipv6-addressing.md
│   ├── switching-concepts.md
│   ├── wireless-principles.md
│   └── configs/
├── 02-network-access/
│   ├── vlans-trunking.md
│   ├── spanning-tree.md
│   ├── etherchannel.md
│   ├── wireless-architecture.md
│   └── configs/
├── 03-ip-connectivity/
│   ├── routing-table.md
│   ├── static-routing.md
│   ├── ospf.md
│   ├── fhrp.md
│   └── configs/
├── 04-ip-services/
│   ├── nat.md
│   ├── dhcp-dns.md
│   ├── ntp-snmp-syslog.md
│   ├── qos.md
│   └── configs/
├── 05-security-fundamentals/
│   ├── acls.md
│   ├── layer2-security.md
│   ├── vpn-ipsec.md
│   ├── wireless-security.md
│   └── configs/
├── 06-automation-programmability/
│   ├── sdn-controllers.md
│   ├── rest-apis.md
│   ├── ansible-terraform.md
│   └── scripts/
└── daily-logs/
    ├── day-01.md
    └── ...
```

### Daily GitHub Ritual (30 min each evening)
1. Open `daily-logs/day-XX.md`
2. Write: *What I learned, CLI commands I used, what confused me, 3 key takeaways*
3. Copy any configs/scripts into the relevant topic folder
4. `git add . && git commit -m "Day XX: <topic>" && git push`

### Note Template (save as `_template.md`)
```markdown
# Topic Name
**Date:** YYYY-MM-DD | **Exam Section:** X.X | **Domain Weight:** X%

## One-Line Definition

## How It Works
- Key point 1
- Key point 2

## Key CLI Commands
```
show ...
configure terminal
 ...
```

## Packet Tracer Lab File
`configs/topic-name.pkt`

## Real-World Analogy

## Exam Gotchas

## show/debug Commands to Verify
```

---

## 🛠️ Tools You Need (Set Up on Day 0)

| Tool | Purpose | Download |
|------|---------|----------|
| **Cisco Packet Tracer** | Free network simulator for all labs | netacad.com (free account) |
| **GNS3** (optional) | More realistic IOS simulation | gns3.com |
| **PuTTY / iTerm** | Terminal for CLI practice | |
| **Wireshark** | Packet capture and analysis | wireshark.org |
| **Git + VS Code** | Notes + GitHub push | |
| **Subnetting practice** | subnettingpractice.com | Daily habit |

> **Subnetting tip:** Do 10 subnetting problems every morning before anything else. It becomes muscle memory by Day 10.

---

## 📅 Day-by-Day Study Plan

---

### DAY 1 — Network Components + Topology Architectures
**Exam coverage:** 1.1, 1.2

#### Theory (1.5h)
- **Routers:** Layer 3, route between networks, make forwarding decisions
- **Layer 2 switches:** MAC-based forwarding, VLANs, no routing
- **Layer 3 switches:** routing + switching, inter-VLAN routing
- **Next-gen firewalls (NGFW):** stateful inspection + app awareness + IPS
- **Access points:** wireless client connectivity
- **Controllers (WLC):** centralised management of APs
- **PoE:** power over Ethernet — 802.3af (15.4W), 802.3at (30W), 802.3bt (60W/90W)
- **Topology architectures:**
  - Two-tier (collapsed core): access + core — small/medium networks
  - Three-tier: access + distribution + core — enterprise
  - Spine-leaf: every leaf connects to every spine — data centre
  - WAN: connects geographically separate sites
  - SOHO: single router/switch/AP combo

#### Hands-On: Packet Tracer (2h)
```
Lab: Build a three-tier topology
- 2 Core switches (Layer 3)
- 2 Distribution switches
- 4 Access switches
- 8 PCs
- Label each device role
- Connect them with correct cable types
- Save as: 01-network-fundamentals/configs/three-tier-topology.pkt
```

#### GitHub Note to Write
- `01-network-fundamentals/osi-tcp-ip-models.md` — device roles table, topology diagrams description, PoE standards

---

### DAY 2 — TCP vs UDP + Cabling + Interface Issues
**Exam coverage:** 1.3, 1.4, 1.5

#### Theory (1.5h)
- **TCP:** connection-oriented, reliable, ordered, flow control, 3-way handshake (SYN → SYN-ACK → ACK)
- **UDP:** connectionless, fast, no guarantee — used for DNS, DHCP, video streaming
- **Cabling types:**
  - Single-mode fiber (SMF): long distance (km), laser, yellow jacket
  - Multimode fiber (MMF): short distance (100–500m), LED, orange/aqua jacket
  - Copper (UTP): Cat5e/Cat6, max 100m, cost-effective
- **Interface issues to recognise:**
  - Collisions: duplex mismatch (full vs half duplex)
  - CRC errors: bad cable or NIC
  - Input/output errors: cable noise, bad port
  - Speed mismatch: manually set vs auto-negotiated

#### Hands-On: CLI (2h)
```
Lab: Connect to a switch in Packet Tracer, practice show commands:
```
```cisco
! Verify interface status
show interfaces GigabitEthernet0/1
show interfaces status
show interfaces counters errors

! Look for:
! - "input errors" → cabling issue
! - "CRC" → physical layer problem
! - Duplex/speed mismatch → manual config needed

! Fix duplex/speed mismatch
interface GigabitEthernet0/1
 duplex full
 speed 1000
 no shutdown
```

#### GitHub Note to Write
- `01-network-fundamentals/osi-tcp-ip-models.md` — TCP vs UDP comparison table, cabling comparison, error types and causes

---

### DAY 3 — IPv4 Addressing and Subnetting (Part 1)
**Exam coverage:** 1.6, 1.7

#### Theory (1.5h)
- **IPv4 structure:** 32-bit, 4 octets, dotted decimal
- **Subnet mask:** determines network vs host portion
- **CIDR notation:** /24 = 255.255.255.0
- **Subnetting formula:** 2^n hosts (subtract 2 for network/broadcast), 2^n subnets
- **Private ranges (RFC 1918):**
  - 10.0.0.0/8 (Class A private)
  - 172.16.0.0/12 (Class B private)
  - 192.168.0.0/16 (Class C private)
- **Special addresses:** loopback (127.0.0.1), APIPA (169.254.x.x)

#### Hands-On: Subnetting Practice (2h)
```
Given: 192.168.10.0/24
Task: Create 4 subnets, each supporting 50 hosts

Solution steps:
1. Need 4 subnets → borrow 2 bits (2^2 = 4)
2. New prefix: /26 (255.255.255.192)
3. Block size: 256 - 192 = 64
4. Subnets:
   192.168.10.0/26   → hosts .1–.62,   broadcast .63
   192.168.10.64/26  → hosts .65–.126, broadcast .127
   192.168.10.128/26 → hosts .129–.190, broadcast .191
   192.168.10.192/26 → hosts .193–.254, broadcast .255

Practice 20 more at: subnettingpractice.com
```

```cisco
! Configure IPv4 on a router interface
interface GigabitEthernet0/0
 ip address 192.168.10.1 255.255.255.192
 description LAN-Subnet-1
 no shutdown

! Verify
show ip interface brief
show ip interface GigabitEthernet0/0
ping 192.168.10.62
```

#### GitHub Note to Write
- `01-network-fundamentals/ipv4-subnetting.md` — subnetting method, formula, private ranges, worked examples

---

### DAY 4 — IPv6 Addressing
**Exam coverage:** 1.8, 1.9

#### Theory (1.5h)
- **IPv6 structure:** 128-bit, 8 groups of 4 hex digits separated by colons
- **Shortening rules:** omit leading zeros, :: for longest consecutive zero groups (once only)
- **Address types:**
  - Global Unicast (GUA): 2000::/3 — routable on internet
  - Unique Local: FC00::/7 — like private IPv4
  - Link Local: FE80::/10 — mandatory on every interface, not routed
  - Multicast: FF00::/8 — one-to-many
  - Anycast: assigned to multiple devices, nearest responds
- **EUI-64:** derives interface ID from 48-bit MAC (insert FFFE in middle, flip 7th bit)
- **Neighbour Discovery Protocol (NDP):** replaces ARP, uses ICMPv6

#### Hands-On: Packet Tracer (2h)
```cisco
! Configure IPv6 on router
ipv6 unicast-routing

interface GigabitEthernet0/0
 ipv6 address 2001:DB8:ACAD:1::1/64
 ipv6 address FE80::1 link-local
 no shutdown

! EUI-64 (router generates host portion from MAC)
interface GigabitEthernet0/1
 ipv6 address 2001:DB8:ACAD:2::/64 eui-64
 no shutdown

! Verify
show ipv6 interface brief
show ipv6 interface GigabitEthernet0/0
ping ipv6 2001:DB8:ACAD:1::2
```

#### GitHub Note to Write
- `01-network-fundamentals/ipv6-addressing.md` — address type table, EUI-64 method, NDP vs ARP comparison

---

### DAY 5 — Wireless Principles + Switching Concepts
**Exam coverage:** 1.11, 1.12, 1.13

#### Theory (1.5h)
- **Wireless:**
  - Non-overlapping channels: 2.4GHz → channels 1, 6, 11; 5GHz → many non-overlapping
  - SSID: wireless network name (can be same across multiple APs)
  - RF signals: 2.4GHz (longer range, more interference), 5GHz (faster, shorter range)
  - Encryption: WPA2 (AES/CCMP), WPA3 (SAE handshake)
- **Virtualization:** VMs (hypervisor), containers (shared OS kernel), VRFs (virtual routing tables)
- **Switching concepts:**
  - MAC learning: source MAC learned when frame arrives on port
  - MAC aging: entries removed after 300 seconds (default)
  - Frame flooding: destination unknown → flood all ports except source
  - Frame switching: destination known → forward to specific port only
  - MAC address table: `show mac address-table`

#### Hands-On: CLI + Packet Tracer (2h)
```cisco
! Switch MAC table verification
show mac address-table
show mac address-table dynamic
show mac address-table address aabb.ccdd.eeff
show mac address-table interface GigabitEthernet0/1

! MAC aging time
mac address-table aging-time 600

! Generate traffic then watch MAC table populate
! (ping between PCs in Packet Tracer)
```

#### GitHub Note to Write
- `01-network-fundamentals/switching-concepts.md` + `wireless-principles.md`

---

### DAY 6 — Verify IP Parameters + Subnetting Review
**Exam coverage:** 1.6, 1.10

#### Theory (30 min)
- **Windows:** `ipconfig /all`, `ipconfig /release`, `ipconfig /renew`
- **macOS/Linux:** `ifconfig`, `ip addr show`, `ip route show`
- **Connectivity tests:** `ping`, `traceroute`/`tracert`, `nslookup`, `netstat`

#### Hands-On: Subnetting Blitz + OS Commands (2h)
```bash
# Linux/macOS commands to know
ip addr show                   # show all interfaces and IPs
ip route show                  # routing table
ping -c 4 8.8.8.8              # test connectivity
traceroute 8.8.8.8             # trace path
nslookup cisco.com             # DNS lookup
netstat -rn                    # routing table (older style)
ss -tuln                       # show listening ports

# Windows equivalents
ipconfig /all
route print
tracert 8.8.8.8
nslookup cisco.com
```

#### Subnetting Drill (1h)
```
Practice these scenarios until fast:
1. Given IP + mask → find network, broadcast, first/last host
2. Given requirements → choose correct prefix length
3. VLSM: assign subnets to networks of different sizes efficiently
   Example: HQ needs 100 hosts, Branch A needs 50, Branch B needs 25, WAN links need 2
   Solution: Start with largest, use 192.168.1.0/25 → /26 → /27 → /30
```

#### GitHub Note to Write
- `01-network-fundamentals/ipv4-subnetting.md` — VLSM section, OS commands table

---

### DAY 7 — WEEK 1 REVIEW + VLANs Introduction
**Exam coverage:** Review + 2.1

#### Morning: Week 1 Review (1.5h)
- Re-read all notes from Days 1–6
- Test yourself: subnet 10.10.0.0/16 into /20 subnets (write them all out)
- Quiz: name 5 interface error types and their causes
- Fix any thin or unclear notes in your GitHub repo

#### Theory: VLANs (1h)
- **VLAN purpose:** segment a switch into multiple broadcast domains
- **Normal range:** 1–1005 (VLAN 1 is default, avoid using it)
- **Extended range:** 1006–4094
- **Access port:** belongs to ONE VLAN — connects end devices
- **Voice VLAN:** data + voice on same port, different VLANs (802.1p priority)
- **Inter-VLAN routing:** router-on-a-stick (subinterfaces) OR Layer 3 switch (SVIs)

#### Hands-On: Packet Tracer (1h)
```cisco
! Create VLANs
vlan 10
 name SALES
vlan 20
 name ENGINEERING
vlan 30
 name MANAGEMENT

! Assign access ports
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 10

! Voice VLAN
interface GigabitEthernet0/2
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 99

! Verify
show vlan brief
show interfaces GigabitEthernet0/1 switchport
```

#### GitHub Note to Write
- `02-network-access/vlans-trunking.md` — VLAN concepts, access port config

---

### DAY 8 — Trunking + Inter-VLAN Routing
**Exam coverage:** 2.1c, 2.2

#### Theory (1.5h)
- **Trunk port:** carries multiple VLANs between switches using 802.1Q tags
- **802.1Q tag:** 4-byte tag inserted into Ethernet frame (VLAN ID field = 12 bits)
- **Native VLAN:** untagged frames on trunk — must match on both ends (default VLAN 1)
- **Router-on-a-stick:** one physical link with subinterfaces, one per VLAN
- **SVI (Switched Virtual Interface):** virtual interface on Layer 3 switch for each VLAN

#### Hands-On: Packet Tracer (2h)
```cisco
! Configure trunk port
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,999

! Verify trunk
show interfaces trunk
show interfaces GigabitEthernet0/1 trunk

! Router-on-a-stick (router side)
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0

! SVI on Layer 3 switch (preferred in modern networks)
ip routing
interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown

! Test inter-VLAN routing — ping from PC in VLAN 10 to PC in VLAN 20
```

#### GitHub Note to Write
- `02-network-access/vlans-trunking.md` — trunking section, 802.1Q diagram, both inter-VLAN methods with comparison

---

### DAY 9 — CDP, LLDP, and EtherChannel
**Exam coverage:** 2.3, 2.4

#### Theory (1.5h)
- **CDP (Cisco Discovery Protocol):** Layer 2, Cisco proprietary, discovers directly connected Cisco devices
  - Sends: device ID, IP, platform, capabilities, port ID
- **LLDP (Link Layer Discovery Protocol):** IEEE 802.1AB, vendor-neutral, used in multi-vendor environments
- **EtherChannel:** bundle multiple physical links into one logical link
  - Protocols: **LACP** (IEEE 802.3ad, open) and **PAgP** (Cisco proprietary)
  - LACP modes: Active (initiate) + Active = forms; Active + Passive = forms; Passive + Passive = does NOT form
  - Load balancing: src-MAC, dst-MAC, src-dst-MAC, src-dst-IP (configured per switch)

#### Hands-On: Packet Tracer (2h)
```cisco
! Enable/verify CDP
cdp run                          ! global enable (default on)
interface GigabitEthernet0/1
 cdp enable
show cdp neighbors
show cdp neighbors detail         ! shows IPs — useful for network discovery

! Enable LLDP
lldp run
interface GigabitEthernet0/1
 lldp transmit
 lldp receive
show lldp neighbors
show lldp neighbors detail

! LACP EtherChannel (Layer 2)
interface range GigabitEthernet0/1-2
 channel-group 1 mode active      ! LACP active
interface Port-channel1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30

! Verify EtherChannel
show etherchannel summary
show etherchannel port-channel
show interfaces Port-channel1
```

#### GitHub Note to Write
- `02-network-access/etherchannel.md` — LACP vs PAgP modes table, load-balance options, verification commands

---

### DAY 10 — Spanning Tree Protocol (STP)
**Exam coverage:** 2.5

#### Theory (1.5h)
- **Problem STP solves:** Layer 2 loops cause broadcast storms → network meltdown
- **Rapid PVST+:** one STP instance per VLAN, Cisco default, fast convergence
- **Root bridge election:** lowest Bridge ID (priority + MAC); default priority 32768
- **Port roles:**
  - Root port (RP): best path to root bridge (one per non-root switch)
  - Designated port (DP): best port on each segment
  - Alternate/Backup port: blocked
- **Port states:** Discarding → Learning → Forwarding (Rapid STP)
- **PortFast:** skip STP states for access ports (direct to forwarding)
- **BPDU Guard:** shut down port if BPDU received on PortFast port
- **Root Guard:** prevent unauthorised root bridge on a port
- **Loop Guard:** prevent alternate port going forwarding if BPDUs stop

#### Hands-On: Packet Tracer (2h)
```cisco
! Set root bridge
spanning-tree vlan 10 priority 4096
spanning-tree vlan 20 priority 4096
! Or use macro:
spanning-tree vlan 10 root primary
spanning-tree vlan 20 root secondary

! PortFast and BPDU Guard on access ports
interface GigabitEthernet0/3
 spanning-tree portfast
 spanning-tree bpduguard enable

! Global PortFast on all access ports (common in production)
spanning-tree portfast default
spanning-tree portfast bpduguard default

! Root Guard on ports facing downstream
interface GigabitEthernet0/1
 spanning-tree guard root

! Verify STP
show spanning-tree
show spanning-tree vlan 10
show spanning-tree vlan 10 detail
show spanning-tree interface GigabitEthernet0/1
```

#### GitHub Note to Write
- `02-network-access/spanning-tree.md` — port role/state table, election rules, guard features diagram

---

### DAY 11 — Wireless Architecture + Device Management
**Exam coverage:** 2.6, 2.7, 2.8, 2.9

#### Theory (1.5h)
- **Wireless AP modes:**
  - Local mode: normal operation, serves clients, tunnels traffic to WLC
  - FlexConnect: can switch traffic locally if WLC unreachable
  - Monitor: passive scanning, rogue detection
  - Sniffer: captures packets for analysis
  - Bridge: point-to-point/mesh wireless links
- **WLC connections:** APs connect via CAPWAP tunnel (UDP 5246/5247)
- **LAG (Link Aggregation):** WLC uplinks bundled for redundancy/bandwidth
- **Device management access:**
  - Console: direct, out-of-band, no network needed
  - Telnet: insecure, plaintext — avoid in production
  - SSH: encrypted CLI access (port 22) — use this
  - HTTP/HTTPS: web GUI management
  - TACACS+: Cisco proprietary, separates auth/authorization/accounting, encrypts entire packet
  - RADIUS: standard, encrypts only password, combines auth+authorization

#### Hands-On: CLI (2h)
```cisco
! Configure SSH access (do this on every device)
hostname SW1
ip domain-name lab.local
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3

! Create local user
username admin privilege 15 secret StrongP@ss123

! VTY lines (remote access)
line vty 0 15
 login local
 transport input ssh
 exec-timeout 10 0

! Disable Telnet
line vty 0 15
 transport input ssh      ! only SSH allowed

! Management VLAN with SVI
interface vlan 99
 ip address 10.99.99.1 255.255.255.0
 no shutdown

! Verify SSH
show ip ssh
show ssh
```

#### GitHub Note to Write
- `02-network-access/wireless-architecture.md` — AP mode table, CAPWAP, TACACS+ vs RADIUS comparison

---

### DAY 12 — Network Access Review + Lab Day
**Exam coverage:** 2.0 domain review

#### Morning: Review (1.5h)
- Re-read all Domain 2 notes
- Quiz yourself: LACP modes that form/don't form, STP port roles, trunk vs access port

#### Lab Challenge (2h) — Build this complete topology in Packet Tracer:
```
Topology:
- 2x Core L3 switches (HSRP for redundancy)
- 2x Access switches
- VLANs: 10 (Data), 20 (Voice), 30 (Management), 99 (Native)
- EtherChannel between core switches (LACP)
- Trunks between all switches
- PortFast + BPDU Guard on all access ports
- Root bridge: Core-SW1 for all VLANs
- SSH configured on all devices
- PCs in each VLAN can ping each other (inter-VLAN routing via SVIs)

Verification checklist:
[ ] show vlan brief — all VLANs exist
[ ] show interfaces trunk — correct VLANs allowed
[ ] show spanning-tree — correct root bridge
[ ] show etherchannel summary — Po1 is up
[ ] ping across VLANs succeeds
[ ] SSH from PC to switch works
```

#### GitHub Note to Write
- `daily-logs/day-12.md` — lab challenges faced, solutions found, commands that helped

---

### DAY 13 — Routing Table + How Routers Forward Packets
**Exam coverage:** 3.1, 3.2

#### Theory (1.5h)
- **Routing table components:**
  - Protocol code: C (connected), S (static), O (OSPF), R (RIP), * (default)
  - Prefix + mask: the destination network
  - Administrative distance (AD): trustworthiness of source (lower = better)
    - Connected: 0, Static: 1, OSPF: 110, RIP: 120, EIGRP: 90, eBGP: 20
  - Metric: cost within a routing protocol (hops, bandwidth, etc.)
  - Next hop: where to send the packet
  - Outgoing interface: physical interface to use
  - Gateway of last resort: default route — used when no specific match
- **Forwarding decision (in order):**
  1. Longest prefix match wins (most specific route)
  2. If tie, lowest AD wins
  3. If tie (same protocol), lowest metric wins
  4. Load balance if metric equal (ECMP)

#### Hands-On: CLI (2h)
```cisco
! Examine a real routing table
show ip route
show ip route 192.168.10.0
show ip route summary

! Understand each line:
! O  192.168.20.0/24 [110/2] via 10.0.0.2, 00:10:15, GigabitEthernet0/0
!  ^  ^              ^ ^    ^ ^             ^          ^
!  |  prefix/len     | |    | next-hop      timestamp  interface
!  protocol         AD metric

! Test longest prefix match
! Route 0.0.0.0/0 (default) vs 10.0.0.0/8 vs 10.1.1.0/24
! Packet to 10.1.1.5 → uses /24 (most specific)
! Packet to 10.2.2.5 → uses /8
! Packet to 8.8.8.8  → uses default /0

show ip route longer-prefixes 10.0.0.0 255.0.0.0
```

#### GitHub Note to Write
- `03-ip-connectivity/routing-table.md` — AD table, routing table anatomy diagram, longest prefix match examples

---

### DAY 14 — Static Routing
**Exam coverage:** 3.3

#### Theory (1.5h)
- **Static route types:**
  - Network route: specific destination network
  - Default route: 0.0.0.0/0 — gateway of last resort
  - Host route: /32 for single host
  - Floating static: higher AD than dynamic route — acts as backup
- **Fully specified vs recursive:**
  - Next-hop IP only: recursive lookup required
  - Exit interface only: risk of ARP flooding on multi-access networks
  - Both: fully specified — recommended for Ethernet

#### Hands-On: Packet Tracer (2h)
```cisco
! Basic static route
ip route 192.168.20.0 255.255.255.0 10.0.0.2          ! next-hop only
ip route 192.168.20.0 255.255.255.0 GigabitEthernet0/1 10.0.0.2  ! fully specified

! Default route
ip route 0.0.0.0 0.0.0.0 203.0.113.1

! Host route
ip route 192.168.30.5 255.255.255.255 10.0.0.2

! Floating static (AD 130 > OSPF 110, used as backup)
ip route 192.168.20.0 255.255.255.0 10.0.0.3 130

! IPv6 static route
ipv6 route 2001:DB8:ACAD:2::/64 2001:DB8:ACAD:1::2
ipv6 route ::/0 2001:DB8:ACAD:1::1   ! IPv6 default route

! Verify
show ip route static
show ip route 0.0.0.0
ping 192.168.20.1 source GigabitEthernet0/0
traceroute 192.168.20.1
```

#### GitHub Note to Write
- `03-ip-connectivity/static-routing.md` — route type comparison, when to use each, verification commands

---

### DAY 15 — OSPFv2 (Part 1): Concepts + Basic Config
**Exam coverage:** 3.4

#### Theory (1.5h)
- **OSPF key concepts:**
  - Link-state protocol: each router knows full topology (LSDB)
  - Dijkstra's SPF algorithm: calculates shortest path
  - AD = 110, metric = cost (reference bandwidth / interface bandwidth)
  - Default reference bandwidth = 100 Mbps (should be adjusted for GigE)
- **OSPF process:** Discover neighbours → Exchange LSAs → Build LSDB → Run SPF → Install routes
- **Neighbour states:** Down → Init → 2-Way → ExStart → Exchange → Loading → Full
- **Router ID:** highest loopback IP, or highest active interface IP, or manually set
- **Neighbour requirements:** same area, subnet, hello/dead timers, MTU, auth, stub flags

#### Hands-On: Packet Tracer (2h)
```cisco
! Basic OSPFv2 single-area configuration
router ospf 1
 router-id 1.1.1.1
 auto-cost reference-bandwidth 1000    ! adjust for GigE networks
 network 192.168.10.0 0.0.0.255 area 0
 network 10.0.0.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/1  ! don't send OSPF on LAN-facing ports

! Verify neighbours and routes
show ip ospf neighbor
show ip ospf neighbor detail
show ip ospf database
show ip route ospf
show ip ospf interface GigabitEthernet0/0
show ip ospf
```

#### GitHub Note to Write
- `03-ip-connectivity/ospf.md` — neighbour states table, OSPF process diagram, configuration template

---

### DAY 16 — OSPFv2 (Part 2): DR/BDR + Router ID + Verification
**Exam coverage:** 3.4

#### Theory (1.5h)
- **DR/BDR election (broadcast networks):**
  - DR (Designated Router): reduces OSPF traffic on multi-access segments
  - BDR (Backup DR): takes over if DR fails
  - Election: highest OSPF priority (default 1) → then highest Router ID
  - Priority 0 = never elected DR/BDR (DROther)
  - Election is non-preemptive (existing DR stays until it fails)
- **Point-to-point links:** no DR/BDR election needed — just two neighbours
- **Passive interfaces:** suppress OSPF hellos on LAN ports (security + efficiency)
- **Default route propagation into OSPF:**
  - `default-information originate` — advertise existing default route
  - `default-information originate always` — advertise even without default route

#### Hands-On: Packet Tracer (2h)
```cisco
! Set OSPF router priority (do this before neighbour forms)
interface GigabitEthernet0/0
 ip ospf priority 100    ! highest priority = DR
 ip ospf cost 10         ! manual cost override

! Verify DR/BDR
show ip ospf neighbor
! Look for: FULL/DR, FULL/BDR, FULL/DROTHER, 2WAY/DROTHER

! Propagate default route
ip route 0.0.0.0 0.0.0.0 203.0.113.1   ! need a default first
router ospf 1
 default-information originate

! Clear OSPF process (forces re-election)
clear ip ospf process          ! in production — use with caution!

! Point-to-point — no DR election
interface Serial0/0
 ip ospf network point-to-point
```

#### GitHub Note to Write
- `03-ip-connectivity/ospf.md` — DR/BDR section, priority rules, show output interpretation

---

### DAY 17 — First Hop Redundancy Protocols (FHRP)
**Exam coverage:** 3.5

#### Theory (1.5h)
- **Problem:** if the default gateway router fails, hosts lose connectivity
- **FHRP solution:** multiple routers share a virtual IP/MAC, hosts point to virtual IP
- **HSRP (Hot Standby Router Protocol):** Cisco proprietary
  - Active router: forwards traffic
  - Standby router: ready to take over
  - Virtual IP and MAC (0000.0C07.ACxx where xx = HSRP group)
  - Priority: highest wins (default 100), preempt needed to take over
  - States: Initial → Learn → Listen → Speak → Standby → Active
- **VRRP (Virtual Router Redundancy Protocol):** IEEE standard (RFC 5798), similar to HSRP
- **GLBP (Gateway Load Balancing Protocol):** Cisco proprietary, actual load balancing

#### Hands-On: Packet Tracer (2h)
```cisco
! HSRP configuration — Active router (R1)
interface GigabitEthernet0/0
 ip address 192.168.10.2 255.255.255.0
 standby version 2
 standby 1 ip 192.168.10.1    ! virtual IP — this is what PCs use as gateway
 standby 1 priority 110        ! higher than default 100 → Active
 standby 1 preempt             ! reclaim Active role if it comes back up
 standby 1 track GigabitEthernet0/1 30  ! decrement priority 30 if WAN fails

! HSRP — Standby router (R2)
interface GigabitEthernet0/0
 ip address 192.168.10.3 255.255.255.0
 standby version 2
 standby 1 ip 192.168.10.1    ! same virtual IP
 standby 1 priority 100        ! lower = standby
 standby 1 preempt

! PC default gateway = 192.168.10.1 (virtual IP)

! Verify HSRP
show standby
show standby brief
show standby GigabitEthernet0/0
```

#### GitHub Note to Write
- `03-ip-connectivity/fhrp.md` — HSRP states, HSRP vs VRRP vs GLBP table, preempt importance

---

### DAY 18 — IP Connectivity Review + Complex Lab
**Exam coverage:** 3.0 domain review

#### Morning: Review (1.5h)
- Re-read Domain 3 notes
- Draw from memory: OSPF neighbour state machine
- Quiz: What wins — AD 110 route or AD 120 route to same network?

#### Lab Challenge (2h) — Full routing topology:
```
Topology:
- R1 (OSPF Area 0) — R2 (OSPF Area 0) — R3 (OSPF Area 0)
- Each router has 2 LAN subnets (use /26 subnets from 10.10.0.0/16)
- R2 connects to "internet" (static default route → ISP)
- R2 propagates default into OSPF
- HSRP on R1/R2 for LAN redundancy (virtual GW for PCs)
- Floating static backup on R1 (AD 130) in case OSPF fails

Verification checklist:
[ ] show ip ospf neighbor — all Full/DR, Full/P2P
[ ] show ip route — OSPF routes with O prefix
[ ] show ip route 0.0.0.0 — default received via OSPF OE2
[ ] show standby brief — one Active, one Standby
[ ] Ping from PC on R1's LAN to PC on R3's LAN
[ ] Shut R1 WAN link — HSRP fails over, floating static activates
```

#### GitHub Note to Write
- `daily-logs/day-18.md` — topology diagram, issues encountered, solutions, lessons learned

---

### DAY 19 — NAT (Network Address Translation)
**Exam coverage:** 4.1

#### Theory (1.5h)
- **Why NAT:** conserves IPv4 addresses, hides internal topology
- **NAT types:**
  - Static NAT: one-to-one mapping (inside local → inside global), always active
  - Dynamic NAT: pool of public IPs, first-come-first-served, no mapping until needed
  - PAT (Port Address Translation / NAT Overload): many-to-one, port number differentiates flows
- **NAT terminology:**
  - Inside local: private IP of internal host
  - Inside global: public IP representing internal host
  - Outside local: IP of external host as seen from inside
  - Outside global: actual IP of external host
- **Common NAT issue:** hairpinning (internal host reaches external IP that maps back to internal server)

#### Hands-On: Packet Tracer (2h)
```cisco
! Static NAT (one server always reachable from internet)
ip nat inside source static 192.168.10.100 203.0.113.100

! Dynamic NAT with pool
ip nat pool PUBLIC_POOL 203.0.113.10 203.0.113.20 netmask 255.255.255.0
access-list 1 permit 192.168.0.0 0.0.255.255
ip nat inside source list 1 pool PUBLIC_POOL

! PAT / NAT Overload (most common — home router style)
access-list 1 permit 192.168.0.0 0.0.255.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload

! Mark interfaces
interface GigabitEthernet0/0
 ip nat outside          ! facing internet
interface GigabitEthernet0/1
 ip nat inside           ! facing LAN

! Verify NAT
show ip nat translations
show ip nat translations verbose
show ip nat statistics
debug ip nat            ! careful in production!
clear ip nat translation *
```

#### GitHub Note to Write
- `04-ip-services/nat.md` — NAT type comparison, terminology diagram, PAT config template

---

### DAY 20 — DHCP, DNS, NTP, SNMP, Syslog
**Exam coverage:** 4.2, 4.3, 4.4, 4.5, 4.6

#### Theory (1.5h)
- **DHCP:** DORA process (Discover→Offer→Request→Acknowledge), lease time
  - DHCP relay (`ip helper-address`) — forwards broadcasts to DHCP server on another subnet
- **DNS:** resolves names to IPs, hierarchical (root → TLD → authoritative)
- **NTP:** synchronises time, stratum levels (0=atomic clock, 1=direct, 2+=via network)
  - Critical for logs, certificates, OSPF, security events
- **SNMP:** network monitoring, versions: v1 (no security), v2c (community strings), v3 (auth+encryption)
  - MIB (Management Information Base): structured tree of OIDs
  - Trap: device-initiated alert to NMS
- **Syslog:** log messages, 8 severity levels (0=Emergency, 7=Debug), facility codes

#### Hands-On: Packet Tracer (2h)
```cisco
! DHCP server on router
ip dhcp excluded-address 192.168.10.1 192.168.10.10   ! reserve these
ip dhcp pool VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8 8.8.4.4
 lease 7                            ! 7 days

! DHCP relay (on interface facing clients, when server is remote)
interface GigabitEthernet0/1
 ip helper-address 10.0.0.100       ! DHCP server IP

! DHCP client (on interface, e.g. towards ISP)
interface GigabitEthernet0/0
 ip address dhcp

! NTP client
ntp server 216.239.35.0             ! Google NTP
ntp server 216.239.35.4
show ntp status
show ntp associations

! Syslog to external server
logging host 10.0.0.200
logging trap warnings                ! send severity 4 (Warnings) and above
logging buffered 16384 debugging

! SNMP v2c
snmp-server community PUBLIC123 RO    ! read-only
snmp-server community PRIVATE456 RW   ! read-write (restrict this!)
snmp-server location "Server Room A"
snmp-server contact "admin@lab.local"
snmp-server enable traps

! Verify
show ip dhcp binding
show ip dhcp pool
show ntp status
show logging
show snmp
```

#### GitHub Note to Write
- `04-ip-services/dhcp-dns.md` + `ntp-snmp-syslog.md` — syslog severity table, SNMP version comparison

---

### DAY 21 — QoS + SSH + FTP/TFTP
**Exam coverage:** 4.7, 4.8, 4.9

#### Theory (1.5h)
- **QoS (Quality of Service):** manage bandwidth when links are congested
  - **Classification:** identify traffic (ACL, DSCP, CoS)
  - **Marking:** tag packets (DSCP/IP Precedence in IP header, CoS in 802.1Q)
  - **Queuing:** prioritise queues (FIFO, WFQ, CBWFQ, LLQ)
  - **Policing:** drop or re-mark traffic exceeding rate (CIR)
  - **Shaping:** delay traffic to smooth rate (buffered, not dropped)
  - **Congestion avoidance:** WRED — drop lower priority packets early
- **TFTP (Trivial FTP):** UDP 69, no auth, no encryption — used for IOS image transfers on LAN
- **FTP:** TCP 20/21, username+password, can be used for config backup

#### Hands-On: Packet Tracer (2h)
```cisco
! Backup IOS image via TFTP
copy flash: tftp:
! Enter source filename, TFTP server IP, destination filename

! Restore config from TFTP
copy tftp: running-config

! Save config
copy running-config startup-config    ! local save
copy running-config tftp:             ! backup to TFTP server

! SSH (recap from Day 11)
show ip ssh
show ssh

! QoS — mark voice traffic with EF (DSCP 46)
class-map match-any VOICE-TRAFFIC
 match protocol rtp audio
access-list 100 permit udp any any range 16384 32767
class-map VOICE
 match access-group 100
policy-map QOS-POLICY
 class VOICE
  priority 512                        ! LLQ — guaranteed 512kbps
 class class-default
  fair-queue                          ! WFQ for everything else
interface GigabitEthernet0/0
 service-policy output QOS-POLICY
```

#### GitHub Note to Write
- `04-ip-services/qos.md` — PHB table (EF, AF, BE, CS), policing vs shaping diagram

---

### DAY 22 — IP Services Review + Security Fundamentals Intro
**Exam coverage:** 4.0 review + 5.1, 5.2

#### Morning: IP Services Review (1.5h)
- Re-read Domain 4 notes
- Quiz: NAT translation table entries, DHCP DORA, syslog severity levels

#### Theory: Security Fundamentals (1h)
- **Threat categories:** malware (virus/worm/trojan/ransomware), phishing, DoS/DDoS, MITM, social engineering
- **Vulnerabilities:** software bugs, misconfigs, weak passwords, unpatched systems
- **Mitigation techniques:** patches, firewalls, IPS, encryption, MFA, principle of least privilege
- **Security program elements:**
  - User awareness: phishing simulation, security training
  - Training: role-specific technical security education
  - Physical access control: badge readers, mantrap, CCTV, clean desk policy
- **Password security elements:**
  - Complexity: length, mixed characters
  - Management: rotation policy, no sharing
  - Alternatives: MFA (something you know + have + are), certificates, biometrics

#### GitHub Note to Write
- `05-security-fundamentals/` — create threat-categories.md with CIA triad, threat taxonomy

---

### DAY 23 — ACLs (Access Control Lists)
**Exam coverage:** 5.6

#### Theory (1.5h)
- **ACL purpose:** filter traffic, classify traffic for QoS/NAT, match routes
- **Standard ACL (1–99, 1300–1999):** matches source IP only — place CLOSE to destination
- **Extended ACL (100–199, 2000–2699):** matches src/dst IP, protocol, ports — place CLOSE to source
- **Named ACL:** same function, easier to edit (can delete individual lines)
- **Processing:** top-down, first match wins, implicit deny all at end
- **Wildcard masks:** inverse of subnet mask (0=must match, 1=ignore)
  - /24 → wildcard 0.0.0.255
  - host 10.1.1.1 → wildcard 0.0.0.0
  - any → wildcard 255.255.255.255

#### Hands-On: Packet Tracer (2h)
```cisco
! Standard ACL — permit only 192.168.10.0/24 to reach 192.168.30.0/24
access-list 10 permit 192.168.10.0 0.0.0.255
access-list 10 deny any               ! explicit (same as implicit deny)
interface GigabitEthernet0/1
 ip access-group 10 out               ! applied outbound toward .30 network

! Extended named ACL — more granular control
ip access-list extended RESTRICT-SALES
 10 permit tcp 192.168.10.0 0.0.0.255 192.168.30.5 0.0.0.0 eq 443
 20 permit icmp 192.168.10.0 0.0.0.255 any
 30 deny ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
 40 permit ip any any
interface GigabitEthernet0/0
 ip access-group RESTRICT-SALES in    ! applied inbound from SALES VLAN

! Edit a named ACL (insert/delete lines)
ip access-list extended RESTRICT-SALES
 no 30                                ! delete line 30
 25 deny udp 192.168.10.0 0.0.0.255 any eq 53   ! insert between 20 and 30

! Verify
show access-lists
show ip interface GigabitEthernet0/0  ! shows applied ACL
show access-lists RESTRICT-SALES      ! shows match counters
```

#### GitHub Note to Write
- `05-security-fundamentals/acls.md` — std vs ext comparison, wildcard mask table, placement rules, numbered sequence

---

### DAY 24 — Layer 2 Security + VPNs
**Exam coverage:** 5.5, 5.7

#### Theory (1.5h)
- **DHCP Snooping:** prevents rogue DHCP servers
  - Trusted ports: uplinks to real DHCP server
  - Untrusted ports: all access ports — drops DHCP Offer/Ack
- **Dynamic ARP Inspection (DAI):** prevents ARP poisoning/spoofing
  - Uses DHCP snooping binding table to validate ARP replies
  - Trusted ports: uplinks; untrusted: access ports
- **Port Security:** restricts which MACs can use a port
  - Modes: shutdown (default), restrict, protect
  - Sticky MAC: dynamically learns and locks MAC
- **IPsec VPN:**
  - Site-to-site: connects two networks permanently (branch to HQ)
  - Remote access: individual users (client VPN, SSL/TLS or IPsec)
  - IKE Phase 1: authenticate peers, establish encrypted tunnel
  - IKE Phase 2: negotiate IPsec SAs for data traffic

#### Hands-On: Packet Tracer (2h)
```cisco
! DHCP Snooping
ip dhcp snooping
ip dhcp snooping vlan 10,20,30
no ip dhcp snooping information option   ! disable option 82 (often needed in PT)
interface GigabitEthernet0/1
 ip dhcp snooping trust          ! uplink to DHCP server or trusted switch
interface GigabitEthernet0/2
 no ip dhcp snooping trust       ! default — access port toward clients

! Dynamic ARP Inspection
ip arp inspection vlan 10,20,30
interface GigabitEthernet0/1
 ip arp inspection trust          ! trusted uplink

! Port Security
interface GigabitEthernet0/3
 switchport mode access
 switchport port-security                          ! enable
 switchport port-security maximum 2                ! allow 2 MACs
 switchport port-security mac-address sticky       ! learn dynamically + save
 switchport port-security violation shutdown       ! shutdown if violated

! Verify Layer 2 security
show ip dhcp snooping
show ip dhcp snooping binding
show ip arp inspection
show ip arp inspection vlan 10
show port-security interface GigabitEthernet0/3
show port-security address
```

#### GitHub Note to Write
- `05-security-fundamentals/layer2-security.md` — three features compared, trusted vs untrusted, violation modes

---

### DAY 25 — Wireless Security + AAA + Password Policy
**Exam coverage:** 5.3, 5.4, 5.8, 5.9, 5.10

#### Theory (1.5h)
- **Wireless security protocols:**
  - WPA (Wi-Fi Protected Access): TKIP encryption, weak — avoid
  - WPA2: AES/CCMP encryption, current standard — use this
    - Personal (PSK): pre-shared key — SOHO
    - Enterprise: 802.1X + RADIUS — corporate
  - WPA3: SAE (Simultaneous Authentication of Equals) handshake, resistant to offline attacks
- **AAA concepts:**
  - Authentication: who are you? (username + password)
  - Authorization: what can you do? (privilege levels, commands)
  - Accounting: what did you do? (audit log)
  - TACACS+: three separate steps, full encryption, Cisco — best for device admin
  - RADIUS: combined auth+authoriz, password-only encryption, standard — best for network access
- **Local password config:**
  - `enable secret` (MD5 hashed) vs `enable password` (plaintext) — always use secret
  - `service password-encryption` — encrypts type 7 (weak), but better than nothing
  - Privilege levels 0–15 (15 = full admin)

#### Hands-On: Packet Tracer + GUI (2h)
```cisco
! Local device password hardening
security passwords min-length 12
enable secret StrongEnable@456

! Encrypt stored passwords
service password-encryption

! Create privilege-level users
username netadmin privilege 15 secret Admin@Secure123
username readonly privilege 1 secret Read@Only456

! Login security — block after failed attempts
login block-for 120 attempts 3 within 60   ! block 2min after 3 fails in 60s
login quiet-mode access-class MGMT-ONLY
login on-failure log
login on-success log

! Banner (legal warning)
banner motd # Authorised access only. All activity is monitored. #

! Wireless WPA2 PSK via GUI:
! WLC GUI → WLANs → Create New → Security tab → Layer 2: WPA+WPA2
! → WPA2 Policy: enabled → AES → PSK: enter passphrase

show users                    ! currently logged in users
show privilege                ! current privilege level
```

#### GitHub Note to Write
- `05-security-fundamentals/wireless-security.md` — WPA/WPA2/WPA3 comparison, AAA section, login hardening template

---

### DAY 26 — Security Review + Full Security Lab
**Exam coverage:** 5.0 domain review

#### Morning: Review (1.5h)
- Re-read all Domain 5 notes
- Quiz: DHCP snooping trusted vs untrusted, TACACS+ vs RADIUS differences, ACL placement rules

#### Lab Challenge (2h) — Secure a complete campus network:
```
Starting point: basic campus topology with VLANs, trunks, routing

Tasks to complete:
1. ACL: Block VLAN 10 (Guest) from accessing VLAN 20 (Finance), allow internet
2. ACL: Permit only SSH (port 22) to management VLAN 99 from any source
3. DHCP Snooping: enable on all VLANs, trust only uplinks
4. DAI: enable on all VLANs
5. Port Security: max 2 MACs, sticky, shutdown on all access ports
6. SSH hardening: RSA 2048, SSH v2 only, 3 retries, 60s timeout
7. Login block: 3 failures in 60s → block for 120s
8. Local users: admin (priv 15), monitor (priv 1)
9. Syslog: severity warnings to 10.0.99.200

Verification checklist:
[ ] show access-lists — match counters incrementing
[ ] show ip dhcp snooping — all access ports untrusted
[ ] show port-security — all access ports configured
[ ] SSH from monitor user — limited commands only
[ ] Attempt bad password 3x — login blocked
```

#### GitHub Note to Write
- `daily-logs/day-26.md` — lab challenges, lessons, security hardening template to reuse

---

### DAY 27 — SDN, Controller-Based Networks, and AI in Networking
**Exam coverage:** 6.1, 6.2, 6.3, 6.4

#### Theory (1.5h)
- **Automation impact:** consistent configs, faster deployment, reduced human error, scalability
- **Traditional vs controller-based:**
  - Traditional: distributed control plane (each device decides independently)
  - Controller-based: centralised control plane, devices just forward (separation of planes)
- **SDN architecture:**
  - Underlay: physical network infrastructure (routers, switches, physical links)
  - Overlay: virtual network on top (tunnels, VXLAN, GRE)
  - Fabric: combined underlay + overlay + controller (e.g. ACI, APIC, Cisco SDA)
  - Northbound API: between controller and applications (REST/JSON — you write scripts against this)
  - Southbound API: between controller and devices (OpenFlow, NETCONF, RESTCONF, OpFlex)
- **AI in networking:**
  - Predictive AI: ML models predict failures, anomalies, traffic patterns
  - Generative AI: natural language config, RCA, chatbot troubleshooting (Cisco AI Assistant)
  - Used in: Cisco Catalyst Center AI Analytics, ThousandEyes, Meraki Insight

#### Hands-On: Conceptual + Scripting (2h)
```python
# Script that talks to a controller's northbound API
# (Cisco Catalyst Center — uses sandbox from Day 5 of CCNA Auto plan)
import requests, base64

# Authenticate
token_url = "https://sandboxdnac.cisco.com/dna/system/api/v1/auth/token"
creds = base64.b64encode(b"devnetuser:Cisco123!").decode()
headers = {"Authorization": f"Basic {creds}", "Content-Type": "application/json"}
token = requests.post(token_url, headers=headers).json()["Token"]

# Northbound API call — get all network devices
dev_url = "https://sandboxdnac.cisco.com/dna/intent/api/v1/network-device"
headers = {"X-Auth-Token": token}
devices = requests.get(dev_url, headers=headers).json()["response"]
for d in devices:
    print(f"{d['hostname']:20} {d['managementIpAddress']:15} {d['platformId']}")
```

#### GitHub Note to Write
- `06-automation-programmability/sdn-controllers.md` — northbound vs southbound, overlay vs underlay, SDN planes diagram

---

### DAY 28 — REST APIs + Ansible/Terraform + JSON
**Exam coverage:** 6.5, 6.6, 6.7

#### Theory (1.5h)
- **REST API characteristics:**
  - Authentication: Basic, token, API key, OAuth
  - CRUD → HTTP verbs: Create=POST, Read=GET, Update=PUT/PATCH, Delete=DELETE
  - Data encoding: JSON (most common), XML
  - Stateless: each request complete in itself
- **JSON structure:**
  - Objects: `{"key": "value"}` — maps to Python dict
  - Arrays: `["item1", "item2"]` — maps to Python list
  - Types: string, number, boolean, null, object, array
- **Ansible capabilities:**
  - Agentless, idempotent, YAML playbooks
  - `ios_command`, `ios_config` modules for network automation
- **Terraform capabilities:**
  - Declarative IaC, `plan/apply/destroy` workflow
  - State management, providers for Meraki/ACI/AWS

#### Hands-On: JSON Parsing + API (2h)
```python
# Recognise and parse JSON-encoded data (exam 6.7)
import json

# This is what a Cisco API returns — be able to interpret it
api_response = '''
{
  "response": [
    {
      "hostname": "cat_9k_1.abc.inc",
      "managementIpAddress": "10.10.22.70",
      "platformId": "C9KV-UADP-8P",
      "softwareVersion": "17.9.20220318:182713",
      "reachabilityStatus": "Reachable",
      "upTime": "1 days, 6:07:52.00",
      "role": "ACCESS",
      "serialNumber": "9AMSYS81Q4C"
    },
    {
      "hostname": "cat_9k_2.abc.inc",
      "managementIpAddress": "10.10.22.71",
      "platformId": "C9KV-UADP-8P",
      "softwareVersion": "17.9.20220318:182713",
      "reachabilityStatus": "Reachable",
      "role": "DISTRIBUTION",
      "serialNumber": "9BMTY2K1P3D"
    }
  ],
  "version": "1.0"
}
'''

data = json.loads(api_response)
devices = data["response"]            # access the array

for device in devices:
    hostname = device["hostname"]     # string value
    ip       = device["managementIpAddress"]
    role     = device["role"]
    print(f"{role:15} | {hostname:30} | {ip}")

# Filter — only ACCESS layer devices
access_devices = [d for d in devices if d["role"] == "ACCESS"]
print(f"\nAccess layer devices: {len(access_devices)}")
```

#### GitHub Note to Write
- `06-automation-programmability/rest-apis.md` — CRUD table, JSON data type examples, Ansible vs Terraform summary

---

### DAY 29 — Full Domain Review + Weak Area Blitz
**Exam coverage:** All domains

#### Morning: Identify Weak Areas (1.5h)
Open your GitHub notes. Score each topic 1–5 (1=shaky, 5=solid).
Focus the rest of today on anything scored 1–2:

**Common weak spots for CCNA:**
- Subnetting speed (keep practicing until < 30 seconds per problem)
- OSPF DR/BDR election rules
- STP port roles and states
- NAT terminology (inside local vs inside global)
- ACL wildcard masks and placement
- HSRP priority + preempt interaction
- Syslog severity levels (memorise 0–7)

#### Afternoon: Commands Blitz (2h)
```cisco
! Must-know verification commands (drill these until automatic)

! Layer 1/2
show interfaces
show interfaces status
show mac address-table
show spanning-tree
show etherchannel summary
show cdp neighbors detail

! VLANs
show vlan brief
show interfaces trunk

! Layer 3 / Routing
show ip route
show ip route summary
show ip ospf neighbor
show ip ospf database
show ip protocols
show standby brief

! Services
show ip dhcp binding
show ip nat translations
show ntp status
show access-lists
show ip ssh
show logging

! Security
show port-security
show ip dhcp snooping binding
show ip arp inspection
```

#### GitHub: Final Polish (1h)
```bash
# Update QUICK-REFERENCE.md with anything you keep forgetting
# Make sure every topic folder has notes
# Tag the repo with topic labels
git add .
git commit -m "Day 29: Full review complete, QUICK-REFERENCE updated"
git push
```

---

### DAY 30 — Mock Exam + Final GitHub Commit
**Exam coverage:** All domains

#### Morning: Simulate Exam (2h)
- Use Boson Ex-Sim, Pearson Test Prep, or Cisco's official practice exam
- Set timer: 120 minutes, 100–120 questions
- Don't check notes — simulate real conditions
- Mark questions you're unsure about

#### Afternoon: Review Wrong Answers (1.5h)
- For every wrong answer, find the topic in your GitHub notes
- Add a "Correction" section with what you got wrong and why
- If a topic has no notes, add them now

#### Evening: Pre-Exam Checklist + Final Commit (30 min)
```bash
# Final GitHub commit
cd ccna-enterprise-notes
echo "## Exam Day Ready — $(date)" >> README.md
git add .
git commit -m "Day 30: Mock exam complete, notes finalised"
git push

# Create a GitHub Release
# On GitHub.com: Releases → Draft new release → v1.0.0 → "Exam Ready"
# This marks the milestone — your second brain is complete
```

**Night before exam:**
- Sleep 8 hours — more valuable than last-minute cramming
- Know your exam centre location
- Have your Cisco login ready
- Bring valid government ID

---

## 📊 Domain Coverage Summary

| Domain | Weight | Days | Key Skills |
|--------|--------|------|-----------|
| 1.0 Network Fundamentals | 20% | 1–6 | Subnetting, IPv6, switching |
| 2.0 Network Access | 20% | 7–12 | VLANs, STP, EtherChannel, wireless |
| 3.0 IP Connectivity | 25% | 13–18 | Routing table, OSPF, HSRP |
| 4.0 IP Services | 10% | 19–21 | NAT, DHCP, QoS |
| 5.0 Security Fundamentals | 15% | 22–26 | ACLs, Layer 2 security, VPN |
| 6.0 Automation | 10% | 27–28 | SDN, REST APIs, JSON |
| Review & Integration | — | 6, 12, 18, 22, 26, 29, 30 | — |

---

## 🔑 Essential Resources

| Resource | Purpose | URL/Note |
|----------|---------|----------|
| **Cisco Packet Tracer** | All switching/routing labs | netacad.com — free |
| **DevNet Sandbox** | Live Catalyst Center, IOS XE | developer.cisco.com/sandbox |
| **SubnettingPractice.com** | Daily subnetting drills | Do 10 per day minimum |
| **Cisco Command Reference** | IOS CLI documentation | cisco.com |
| **Boson Ex-Sim** | Best practice exam simulator | boson.com (paid, worth it) |
| **Wireshark** | Packet analysis for protocol understanding | wireshark.org |

---

## 💡 GitHub Second Brain — Pro Tips

1. **Commit every day** — the git log becomes your study journal
2. **Save working Packet Tracer files** to the `configs/` folders — you can reopen and experiment anytime
3. **Use GitHub Issues** as a flashcard system — open an Issue for each concept you're unsure of, close it when you can explain it out loud without notes
4. **Add a subnetting log** — every day, record how long your subnets took. Watch it drop from 3 minutes to 20 seconds
5. **Create a `QUICK-REFERENCE.md`** at root level — your cheat sheet for commands you keep forgetting, updated daily

```yaml
# .github/workflows/check-notes.yml
# Optional: auto-check that every day has a log entry
name: Study Log Check
on:
  push:
    paths: ['daily-logs/**']
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Count daily logs
        run: |
          count=$(ls daily-logs/*.md | wc -l)
          echo "Daily logs committed: $count/30"
```

---

*The network doesn't care about theory — it cares about what you can actually configure and troubleshoot. Build the labs, commit the notes, and by Day 30 you'll think like a network engineer.*
