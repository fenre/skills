# Switching & Layer 2 Deep Reference

## VLANs

### Fundamentals
- IEEE 802.1Q encapsulation; 12-bit VLAN ID field = 4094 usable VLANs (0 and 4095 reserved)
- VLAN 1 = default VLAN (cannot be deleted; carries CDP, VTP, DTP, PAgP by default)
- Reserved VLANs: 1002-1005 (FDDI/Token Ring legacy); 3968-4047 (internal use on some platforms)
- Normal range: 1-1005; Extended range: 1006-4094 (requires VTP transparent or VTPv3)

### Trunking (802.1Q)
- Inserts 4-byte tag after source MAC: TPID (0x8100) + PCP (3-bit CoS) + DEI (1-bit) + VID (12-bit)
- Native VLAN: frames sent untagged on trunk (default VLAN 1; best practice: change to unused VLAN)
- ISL: Cisco legacy encapsulation; fully deprecated; do not use
- DTP (Dynamic Trunking Protocol): auto-negotiates trunk; best practice: disable with `switchport nonegotiate`

### VLAN Configuration (Cisco IOS-XE)
```
vlan 100
 name DATA
vlan 200
 name VOICE
!
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 100
 switchport voice vlan 200
 spanning-tree portfast
 spanning-tree bpduguard enable
!
interface GigabitEthernet1/0/24
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 100,200,300
 switchport nonegotiate
```

### VTP (VLAN Trunking Protocol)
- v1/v2: server/client/transparent modes; revision number propagation can wipe VLANs
- v3: adds primary server concept, extended VLAN support, per-instance control
- Best practice: use VTP transparent mode or VTP off (NX-OS default) to avoid accidental VLAN deletion

---

## Spanning Tree Protocol (STP) Family

### STP (802.1D) — Legacy
- Convergence: 30-50 seconds (Listening 15s → Learning 15s → Forwarding)
- Port states: Blocking, Listening, Learning, Forwarding, Disabled
- Port roles: Root, Designated, Non-Designated (Blocked)
- Root Bridge election: Lowest Bridge ID (priority:MAC); default priority 32768
- Path cost: based on link speed (10M=100, 100M=19, 1G=4, 10G=2)

### RSTP (802.1w) — Rapid Spanning Tree
- Convergence: 1-6 seconds (proposal/agreement mechanism)
- Port states: Discarding, Learning, Forwarding (3 states vs 5)
- Port roles: Root, Designated, Alternate (backup for root port), Backup (backup for designated)
- Proposal/Agreement: edge ports and P2P links converge instantly without timers
- RSTP is backward compatible with STP (but falls back to slow convergence on STP segments)

### PVST+ / Rapid PVST+
- Cisco proprietary: one STP instance per VLAN
- Allows per-VLAN root bridge placement and load balancing across VLANs
- Rapid PVST+ = RSTP per VLAN (default on modern Catalyst IOS-XE)
- Scale concern: BPDUs per VLAN; keep to <50-100 VLANs on trunks

### MST (802.1s) — Multiple Spanning Tree
- Maps multiple VLANs to a smaller number of spanning tree instances (MSTIs)
- MST Region: switches must share same name, revision number, and VLAN-to-instance mapping
- IST (Internal Spanning Tree, Instance 0): carries all VLANs not explicitly mapped
- CIST (Common and Internal Spanning Tree): IST + inter-region spanning tree
- Max 16 instances (0-15) per region
- Interoperates with PVST+ at region boundary via PVST simulation

### MST Configuration (Cisco)
```
spanning-tree mode mst
spanning-tree mst configuration
 name CAMPUS
 revision 1
 instance 1 vlan 100-200
 instance 2 vlan 201-300
!
spanning-tree mst 0 priority 4096
spanning-tree mst 1 priority 0
spanning-tree mst 2 priority 4096
```

### STP Protection Features
| Feature | Purpose | Applied To |
|---------|---------|-----------|
| PortFast | Skip Listening/Learning on access ports | Edge (host) ports |
| BPDU Guard | Disable port if BPDU received | PortFast ports |
| BPDU Filter | Suppress BPDUs (dangerous; use carefully) | Edge ports (global) |
| Root Guard | Prevent port from becoming root port | Designated ports toward edge |
| Loop Guard | Prevent alternate/root port from going forwarding on unidirectional link | Non-designated ports |
| UplinkFast | Fast failover for access layer (legacy; replaced by RSTP) | Access switches |
| BackboneFast | Detect indirect failures (legacy; replaced by RSTP) | All switches |

### STP Best Practices
1. Use Rapid PVST+ or MST (never legacy STP)
2. Explicitly set root bridge: `spanning-tree vlan X priority 0` (or `root primary`)
3. Enable PortFast + BPDU Guard on all access ports
4. Enable Root Guard on ports facing downstream switches
5. Set native VLAN to an unused VLAN on trunks
6. Prune VLANs on trunks to only those needed
7. Use Layer 3 where possible to reduce STP domain

---

## Port-Channels / Link Aggregation

### LACP (IEEE 802.3ad / 802.1AX)
- **Standard**: IEEE; interoperable across vendors
- **Modes**: Active (initiates) / Passive (responds)
- **Both sides passive** = no channel formed
- **Max members**: 16 (8 active + 8 standby)
- **System priority**: 32768 (default); lower = preferred
- **Port priority**: 32768 (default); lower = selected first when >8 links
- **Rate**: Slow (30s) or Fast (1s) LACPDU interval

### PAgP (Port Aggregation Protocol)
- **Cisco proprietary**; modes: Desirable (initiates) / Auto (responds)
- Use LACP for multi-vendor environments

### Hashing / Load Balancing
| Method | Command (Cisco) | Use Case |
|--------|----------------|----------|
| src-mac | `port-channel load-balance src-mac` | L2 traffic |
| dst-mac | `port-channel load-balance dst-mac` | L2 traffic |
| src-dst-mac | `port-channel load-balance src-dst-mac` | L2 mixed |
| src-ip | `port-channel load-balance src-ip` | L3 routed |
| dst-ip | `port-channel load-balance dst-ip` | L3 routed |
| src-dst-ip | `port-channel load-balance src-dst-ip` | L3 mixed (common) |
| src-dst-ip-port | — (NX-OS / enhanced) | Best distribution |

### Multi-Chassis LAG (MLAG)
| Vendor | Technology | Protocol |
|--------|-----------|----------|
| Cisco (Nexus) | vPC (Virtual Port-Channel) | Proprietary; peer-link + peer-keepalive |
| Cisco (Catalyst) | StackWise / SVL (StackWise Virtual) | Proprietary; switch stacking |
| Arista | MLAG | Proprietary; peer-link + VLAN |
| Juniper | MC-LAG / Virtual Chassis | Standards-based MC-LAG or VC |
| Aruba | VSX (Virtual Switching Extension) | Proprietary; keepalive + ISL |

### Port-Channel Config Example (Cisco)
```
interface range GigabitEthernet1/0/1-2
 channel-group 1 mode active
!
interface Port-channel1
 switchport mode trunk
 switchport trunk allowed vlan 100,200
```

---

## VXLAN / EVPN

### VXLAN (Virtual Extensible LAN)
- **RFC 7348**; encapsulates L2 frames in UDP (port 4789)
- **VNI**: 24-bit VXLAN Network Identifier = 16,777,216 segments (vs 4094 VLANs)
- **VTEP**: VXLAN Tunnel Endpoint; performs encap/decap
- **Overhead**: 50 bytes (outer Ethernet 14 + outer IP 20 + UDP 8 + VXLAN header 8)
- **MTU**: Underlay requires at least 1550 (typically set to 9214 for jumbo frames)

### EVPN (Ethernet VPN)
- **RFC 7432**; BGP address family `l2vpn evpn`
- Control-plane MAC learning replaces data-plane flood-and-learn
- Provides ARP suppression, reducing broadcast in overlay

### EVPN Route Types
| Type | Name | Purpose |
|------|------|---------|
| 1 | Ethernet Auto-Discovery | Multi-homing, aliasing, mass withdraw |
| 2 | MAC/IP Advertisement | MAC and optional IP (ARP suppression) |
| 3 | Inclusive Multicast | BUM traffic replication tree |
| 4 | Ethernet Segment | DF election for multi-homing |
| 5 | IP Prefix | L3 VPN prefix advertisement |

### VXLAN EVPN Deployment Models
- **Symmetric IRB**: Routing on both ingress and egress VTEP (uses L3 VNI)
- **Asymmetric IRB**: Routing on ingress VTEP only; egress bridges (all VLANs needed everywhere)
- **Centralized routing**: Dedicated border/spine does all inter-VLAN routing
- Symmetric IRB is the standard for modern fabrics (Cisco, Arista, Juniper)

### VXLAN EVPN Config (Cisco NX-OS)
```
feature nv overlay
feature vn-segment-vlan-based
nv overlay evpn

fabric forwarding anycast-gateway-mac 0000.2222.3333

vlan 100
  vn-segment 10100
vlan 200
  vn-segment 10200
vlan 900
  vn-segment 50900

vrf context TENANT-A
  vni 50900
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 10100
    ingress-replication protocol bgp
  member vni 10200
    ingress-replication protocol bgp
  member vni 50900 associate-vrf

interface Vlan100
  vrf member TENANT-A
  ip address 10.1.100.1/24
  fabric forwarding mode anycast-gateway

router bgp 65001
  address-family l2vpn evpn
    retain route-target all
  neighbor 10.0.0.1
    remote-as 65001
    address-family l2vpn evpn
      send-community extended

evpn
  vni 10100 l2
    rd auto
    route-target import auto
    route-target export auto
```

### VXLAN EVPN Config (Arista EOS)
```
vlan 100
   name DATA
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 100 vni 10100
   vxlan vrf TENANT-A vni 50900
!
router bgp 65001
   neighbor SPINE peer group
   neighbor SPINE remote-as 65000
   neighbor SPINE send-community extended
   !
   vlan 100
      rd auto
      route-target both 100:10100
      redistribute learned
   !
   vrf TENANT-A
      rd 1.1.1.1:1
      route-target import evpn 1:50900
      route-target export evpn 1:50900
```

### Multi-Homing (ESI)
- EVPN supports active-active multi-homing via Ethernet Segment Identifier (ESI)
- DF (Designated Forwarder) election ensures no duplicate BUM frames
- Type 1 and Type 4 routes used for multi-homing signaling
- Replaces vPC/MLAG in pure EVPN fabrics (with EVPN multi-homing)
