# WAN Technologies Deep Reference

## MPLS (Multiprotocol Label Switching)

### Fundamentals
- **Ethertype**: 0x8847 (unicast), 0x8848 (multicast)
- **Label**: 20-bit (0-1,048,575); values 0-15 reserved
- **Header**: 32 bits = Label (20) + TC/EXP (3) + S-bit (1) + TTL (8)
- **Reserved labels**: 0=Explicit NULL (IPv4), 1=Router Alert, 2=Explicit NULL (IPv6), 3=Implicit NULL (PHP)
- **PHP (Penultimate Hop Popping)**: Second-to-last router removes label; last router does IP lookup

### Label Distribution Protocol (LDP)
- **Transport**: TCP 646 (session), UDP 646 (discovery via multicast 224.0.0.2)
- **Modes**: Downstream Unsolicited (default Cisco), Downstream on Demand
- **Label allocation**: Independent (default) or Ordered
- **Session**: Targeted LDP for non-adjacent neighbors (used with RSVP-TE backup)

### RSVP-TE (Traffic Engineering)
- **Transport**: IP Protocol 46
- **Messages**: Path, Resv, PathErr, ResvErr, PathTear, ResvTear
- **Creates**: Explicit LSPs with bandwidth reservation
- **FRR**: Facility backup (bypass tunnel) or one-to-one backup (detour)
- **CSPF**: Constrained SPF for path computation using TE database

### MPLS L3VPN
- **RD (Route Distinguisher)**: 8 bytes; makes routes unique in VPNv4/v6 table (NOT a routing attribute)
- **RT (Route Target)**: Extended community; controls import/export between VRFs
- **PE-CE protocols**: Static, BGP (most common), OSPF, EIGRP, RIPv2
- **MP-BGP**: Carries VPNv4/v6 prefixes between PEs using address-family vpnv4/vpnv6
- **Label stack**: Transport label (LDP/RSVP) + VPN label (BGP)

### MPLS L3VPN Config (Cisco IOS-XE)
```
vrf definition CUST-A
 rd 65001:100
 address-family ipv4
  route-target export 65001:100
  route-target import 65001:100
 exit-address-family
!
interface GigabitEthernet0/0
 vrf forwarding CUST-A
 ip address 10.1.1.1 255.255.255.0
!
router bgp 65001
 address-family vpnv4
  neighbor 2.2.2.2 activate
  neighbor 2.2.2.2 send-community extended
 !
 address-family ipv4 vrf CUST-A
  neighbor 10.1.1.2 remote-as 65100
  neighbor 10.1.1.2 activate
```

### MPLS L2VPN Types
| Type | Standard | Use Case |
|------|----------|----------|
| VPWS (Pseudowire) | RFC 4447 | Point-to-point L2 circuit |
| VPLS | RFC 4761/4762 | Multipoint L2 (bridged) |
| EVPN | RFC 7432 | Modern multipoint L2/L3 (replaces VPLS) |
| Atom/EoMPLS | — | Cisco: Ethernet over MPLS |

---

## IPsec VPN

### IKE (Internet Key Exchange)
| Feature | IKEv1 | IKEv2 |
|---------|-------|-------|
| Messages to establish SA | 9 (Main Mode) or 6 (Aggressive) | 4 |
| NAT Traversal | Separate RFC | Built-in |
| EAP Authentication | No | Yes |
| MOBIKE | No | Yes (re-homing) |
| Port | UDP 500 (+ 4500 NAT-T) | UDP 500 (+ 4500 NAT-T) |

### IPsec Protocols
| Protocol | IP Number | Purpose | Encryption | Authentication |
|----------|----------|---------|------------|---------------|
| ESP | 50 | Encrypt + authenticate | Yes | Yes (optional) |
| AH | 51 | Authenticate only | No | Yes |

- **Transport mode**: Encrypts payload only (host-to-host)
- **Tunnel mode**: Encrypts entire original IP packet; adds new IP header (gateway-to-gateway)

### IPsec Transform Sets (common)
| Encryption | Hash | DH Group | Use Case |
|-----------|------|----------|----------|
| AES-256-GCM | SHA-256 (IKE) | 19 (256-bit ECP) or 20 (384-bit) | Modern standard |
| AES-256-CBC | SHA-256 | 14 (2048-bit) | Legacy compatible |
| AES-128-GCM | SHA-256 | 19 | Performance-optimized |

### Cisco IOS-XE IKEv2 VPN Config
```
crypto ikev2 proposal IKEv2-PROP
 encryption aes-cbc-256
 integrity sha256
 group 19
!
crypto ikev2 policy IKEv2-POL
 proposal IKEv2-PROP
!
crypto ikev2 keyring KEYS
 peer REMOTE
  address 203.0.113.1
  pre-shared-key MY-SECRET-KEY
!
crypto ikev2 profile IKEv2-PROF
 match identity remote address 203.0.113.1
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYS
!
crypto ipsec transform-set AES256-SHA esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto map IPSEC-MAP 10 ipsec-isakmp
 set peer 203.0.113.1
 set transform-set AES256-SHA
 set ikev2-profile IKEv2-PROF
 match address CRYPTO-ACL
!
interface GigabitEthernet0/0
 crypto map IPSEC-MAP
```

---

## GRE (Generic Routing Encapsulation)

### Fundamentals
- **IP Protocol**: 47
- **Overhead**: 24 bytes (4 GRE + 20 new IP header) without options
- **Supports**: Multicast, broadcast, non-IP protocols (IPX, AppleTalk)
- **No encryption**: must pair with IPsec for security
- **GRE over IPsec**: most common; GRE provides multicast support, IPsec provides encryption
- **MTU**: Reduce by 24 (GRE) + 52-73 (IPsec) = effective ~1400 bytes typical; set `ip mtu 1400` and `ip tcp adjust-mss 1360`

---

## DMVPN (Dynamic Multipoint VPN)

### Fundamentals
- Cisco proprietary; combines mGRE + NHRP + IPsec + routing protocol
- **Hub-and-spoke initially**; spokes dynamically build direct tunnels

### DMVPN Phases
| Phase | Spoke-to-Spoke | Routing | Data Path |
|-------|---------------|---------|-----------|
| Phase 1 | No | Hub summarizes | All traffic via hub |
| Phase 2 | Yes (direct) | No summarization | Direct after NHRP resolution |
| Phase 3 | Yes (direct) | Hub can summarize | Direct via NHRP redirect/shortcut |

- **NHRP**: Next Hop Resolution Protocol; maps tunnel IP to NBMA (physical) IP
- **Phase 3 preferred**: allows summarization at hub while still enabling spoke-to-spoke

### DMVPN Config (Hub - Cisco IOS-XE)
```
interface Tunnel0
 ip address 172.16.0.1 255.255.255.0
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel key 100
 ip nhrp network-id 1
 ip nhrp map multicast dynamic
 ip nhrp redirect
 ip mtu 1400
 ip tcp adjust-mss 1360
```

### DMVPN Config (Spoke)
```
interface Tunnel0
 ip address 172.16.0.2 255.255.255.0
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel key 100
 ip nhrp network-id 1
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 ip nhrp shortcut
 ip mtu 1400
 ip tcp adjust-mss 1360
```

---

## SD-WAN

### Cisco Catalyst SD-WAN (formerly Viptela)
**Components:**
- **vManage**: Centralized management (NMS), API, monitoring
- **vSmart**: SDN controller; pushes OMP routes and policies
- **vBond**: Orchestrator; initial authentication and discovery
- **vEdge / cEdge**: Edge routers (cEdge = IOS-XE based, preferred)

**Protocols:**
- **OMP (Overlay Management Protocol)**: BGP-like; carries routes, TLOCs, services between vSmart and edges
- **TLOC (Transport Locator)**: {IP, color, encap} tuple identifying a WAN transport
- **BFD**: Probes between edges for path quality (loss, latency, jitter)
- **DTLS/TLS**: Control plane encryption (vManage/vSmart/vBond)
- **IPsec**: Data plane encryption between edges

**Key capabilities:**
- Application-aware routing (AAR): route traffic per-app based on SLA (loss/latency/jitter)
- Centralized policy: data/control/app-route policies pushed from vSmart
- Direct Internet Access (DIA): local breakout at branch
- Cloud OnRamp: optimized paths to SaaS/IaaS
- Multi-topology: segment the network for different tenants/VPNs

### Cato Networks SASE

**Architecture:**
- Cloud-native: no on-premises controllers
- **Cato Socket**: CPE at branch; tunnel to nearest Cato PoP
- **Cato PoP**: Global backbone with full security stack (85+ PoPs)
- **SPACE Engine**: Single Pass Cloud Engine — networking + security in one pass

**Components:**
- SD-WAN (traffic optimization, link aggregation, QoS)
- FWaaS (Firewall as a Service)
- SWG (Secure Web Gateway)
- CASB (Cloud Access Security Broker)
- ZTNA (Zero Trust Network Access)
- IPS, Anti-Malware, DLP — all inline
- Global backbone: private Cato network between PoPs (alternative to MPLS)

**Key differences vs traditional SD-WAN:**
- No on-prem security appliances needed
- All security inspection in the cloud
- Single management plane for networking + security
- Built-in global backbone (no MPLS required)
- Socket deployment: auto-configured, zero-touch

---

## SASE (Secure Access Service Edge)

### Definition (Gartner)
Converges WAN edge networking (SD-WAN) and network security (SWG, CASB, ZTNA, FWaaS) into a single cloud-delivered service.

### SASE Vendors
| Vendor | Approach | Key Differentiator |
|--------|----------|-------------------|
| Cato Networks | Single-vendor cloud-native | Built-in backbone, single SPACE engine |
| Cisco (Umbrella + SD-WAN) | Multi-product integration | Deep network integration, Meraki/Catalyst |
| Palo Alto (Prisma SASE) | Prisma SD-WAN + Prisma Access | Advanced threat prevention |
| Zscaler | ZIA + ZPA + SD-WAN partner | Largest security cloud |
| Fortinet | FortiSASE | FortiGate-based, hardware acceleration |
| VMware / Broadcom | VeloCloud SD-WAN + SASE | Multi-cloud focus |

---

## First Hop Redundancy Protocols (FHRP)

| Protocol | Standard | Virtual MAC | Multicast/Port | Max Groups | Load Balancing |
|----------|----------|-------------|----------------|------------|----------------|
| HSRP v1 | Cisco | 0000.0c07.acXX | 224.0.0.2 / UDP 1985 | 256 | Multiple groups |
| HSRP v2 | Cisco | 0000.0c9f.fXXX | 224.0.0.102 / UDP 2029 | 4096 | Multiple groups |
| VRRP v2 | RFC 3768 | 0000.5e00.01XX | 224.0.0.18 / IP 112 | 255 | None (use GLBP) |
| VRRP v3 | RFC 5798 | 0000.5e00.01XX | 224.0.0.18 / IP 112 | 255 | None |
| GLBP | Cisco | 0007.b400.XXYY | 224.0.0.102 / UDP 3222 | 1024 | Per-host via AVF |

- HSRP: Active/Standby; preempt disabled by default
- VRRP: Master/Backup; preempt enabled by default
- GLBP: Active Virtual Gateway (AVG) assigns virtual MACs to Active Virtual Forwarders (AVFs)
