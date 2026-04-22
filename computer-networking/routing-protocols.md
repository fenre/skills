# Routing Protocols Deep Reference

## BGP (Border Gateway Protocol)

### Fundamentals
- **Type**: Path-vector, EGP (also used as iBGP within AS)
- **Transport**: TCP port 179
- **AD**: eBGP=20, iBGP=200
- **Timers**: Keepalive 60s, Hold 180s (negotiated to lowest)
- **Message types**: OPEN, UPDATE, KEEPALIVE, NOTIFICATION, ROUTE-REFRESH

### BGP Neighbor States (in order)
1. **Idle** — no TCP connection; BGP process initialized
2. **Connect** — TCP SYN sent, waiting for TCP connection
3. **Active** — TCP connection failed; retrying (often means misconfigured neighbor)
4. **OpenSent** — TCP connected; OPEN message sent
5. **OpenConfirm** — OPEN received; parameters accepted; waiting for KEEPALIVE
6. **Established** — session up; exchanging UPDATE messages

### Path Attributes

| Attribute | Category | Type Code | Purpose |
|-----------|----------|-----------|---------|
| ORIGIN | Well-known mandatory | 1 | How route entered BGP (i/e/?) |
| AS_PATH | Well-known mandatory | 2 | Ordered list of AS traversed |
| NEXT_HOP | Well-known mandatory | 3 | Next-hop IP |
| MED | Optional non-transitive | 4 | Suggest entry point to neighbor AS |
| LOCAL_PREF | Well-known discretionary | 5 | Prefer exit within AS (default 100) |
| ATOMIC_AGGREGATE | Well-known discretionary | 6 | Route was aggregated |
| AGGREGATOR | Optional transitive | 7 | AS and router that aggregated |
| COMMUNITY | Optional transitive | 8 | 32-bit tag for policy |
| EXTENDED COMMUNITY | Optional transitive | 16 | 64-bit tag (RT, SOO) |
| LARGE COMMUNITY | Optional transitive | 32 | 96-bit tag (RFC 8092) |
| ORIGINATOR_ID | Optional non-transitive | 9 | Loop prevention in RR |
| CLUSTER_LIST | Optional non-transitive | 10 | RR cluster path |
| WEIGHT | N/A (Cisco-only) | — | Local preference, not advertised |

### Well-Known Communities
- `NO_EXPORT` (0xFFFFFF01) — Do not advertise to eBGP peers
- `NO_ADVERTISE` (0xFFFFFF02) — Do not advertise to any peer
- `NO_EXPORT_SUBCONFED` (0xFFFFFF03) — Do not advertise outside confederation sub-AS
- `NOPEER` (0xFFFFFF04) — Do not advertise to bilateral peers (RFC 3765)

### Route Reflectors
- Solves iBGP full-mesh requirement
- RR reflects routes between clients and non-clients
- Uses ORIGINATOR_ID and CLUSTER_LIST for loop prevention
- Clients peer only with RR; non-clients still need full mesh between themselves
- Design: typically deploy RRs in pairs per cluster for redundancy

### Confederations
- Alternative to RR: splits AS into sub-AS
- Sub-AS uses eBGP between them but preserves iBGP attributes
- AS_CONFED_SEQUENCE and AS_CONFED_SET in AS_PATH
- MED, LOCAL_PREF, NEXT_HOP preserved across sub-AS boundaries

### Address Families
- IPv4 Unicast, IPv6 Unicast, VPNv4, VPNv6, L2VPN EVPN, IPv4/IPv6 Multicast, FlowSpec, MVPN, RT-Constraint
- Activated per neighbor under `address-family` submode

### Cisco IOS/IOS-XE BGP Config Example
```
router bgp 65001
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 neighbor 10.0.0.2 remote-as 65002
 neighbor 10.0.0.2 description eBGP-to-ISP
 neighbor 10.0.0.2 update-source Loopback0
 !
 address-family ipv4 unicast
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 route-map ISP-IN in
  neighbor 10.0.0.2 route-map ISP-OUT out
  network 192.168.0.0 mask 255.255.0.0
 exit-address-family
```

### Juniper Junos BGP Config Example
```
protocols {
    bgp {
        group eBGP-ISP {
            type external;
            peer-as 65002;
            neighbor 10.0.0.2 {
                description "eBGP to ISP";
                import ISP-IN;
                export ISP-OUT;
            }
        }
    }
}
routing-options {
    router-id 1.1.1.1;
    autonomous-system 65001;
}
```

### Route Filtering Tools
- **Prefix-list**: Match exact prefix and mask length
- **AS_PATH ACL / regex**: Match AS path patterns (`^$` = local, `_65001_` = transits AS 65001)
- **Route-map**: Combine match/set; applied per-neighbor
- **Community-list**: Match communities for policy
- **Junos policy-statement**: Equivalent of route-map with term/from/then structure

---

## OSPF (Open Shortest Path First)

### Fundamentals
- **Type**: Link-state, IGP
- **Transport**: IP Protocol 89 (NOT TCP/UDP)
- **Algorithm**: Dijkstra (SPF)
- **AD**: 110
- **Metric**: Cost = Reference BW / Interface BW (default reference 100 Mbps on IOS; 40 Gbps recommended)
- **Multicast**: 224.0.0.5 (AllSPFRouters), 224.0.0.6 (AllDRRouters)

### OSPF Timers (defaults)
| Network Type | Hello | Dead | Wait |
|-------------|-------|------|------|
| Broadcast | 10s | 40s | 40s |
| Point-to-Point | 10s | 40s | 40s |
| NBMA | 30s | 120s | 120s |
| Point-to-Multipoint | 30s | 120s | — |

LSA MaxAge: 3600s (1 hour). LSRefreshTime: 1800s (30 min).

### Network Types
| Type | DR/BDR | Hello | Neighbor Discovery | Use Case |
|------|--------|-------|--------------------|----------|
| Broadcast | Yes | 10s | Automatic | Ethernet |
| Point-to-Point | No | 10s | Automatic | Serial, P2P Ethernet |
| NBMA | Yes | 30s | Manual (`neighbor`) | Frame Relay hub |
| Point-to-Multipoint | No | 30s | Automatic | Frame Relay spoke |
| Point-to-Multipoint NBMA | No | 30s | Manual | Spoke with unequal cost |
| Loopback | No | — | — | /32 always advertised |

Best practice: use `ip ospf network point-to-point` on Ethernet P2P links to avoid DR election.

### DR/BDR Election
1. Highest OSPF priority (default 1; 0 = never DR)
2. Highest Router ID
- Election is non-preemptive: changing priority won't change DR until OSPF restarts

### OSPFv3 (IPv6)
- Runs over IPv6 link-local addresses
- Uses Instance ID for multiple instances per link
- Address families: can carry both IPv4 and IPv6 (RFC 5838)
- Authentication via IPsec (not built-in like OSPFv2)

### Cisco IOS OSPF Config Example
```
router ospf 1
 router-id 1.1.1.1
 auto-cost reference-bandwidth 100000
 passive-interface default
 no passive-interface GigabitEthernet0/0
 network 10.0.0.0 0.0.0.255 area 0
 network 192.168.1.0 0.0.0.255 area 1
 area 1 nssa
```

### Juniper Junos OSPF Config Example
```
protocols {
    ospf {
        reference-bandwidth 100g;
        area 0.0.0.0 {
            interface ge-0/0/0.0;
            interface lo0.0 passive;
        }
        area 0.0.0.1 {
            nssa;
            interface ge-0/0/1.0;
        }
    }
}
```

---

## EIGRP (Enhanced Interior Gateway Routing Protocol)

### Fundamentals
- **Type**: Advanced distance-vector (Cisco called it "hybrid"), IGP
- **Transport**: IP Protocol 88 (NOT TCP/UDP); uses RTP (Reliable Transport Protocol, NOT the audio RTP)
- **AD**: Internal=90, External=170, Summary=5
- **Multicast**: 224.0.0.10
- **Algorithm**: DUAL (Diffusing Update Algorithm)

### Metric Calculation (Classic)
Default formula (K1=1, K2=0, K3=1, K4=0, K5=0):

**metric = 256 * [ (10^7 / minimum_bandwidth_kbps) + (sum_of_delays_in_tens_of_microseconds) ]**

- Bandwidth: minimum along path (in Kbps)
- Delay: cumulative along path (in tens of microseconds)

### Wide Metrics (Named Mode)
- 64-bit values; avoids metric capping for high-speed links
- RIB scale factor: divide by 128 before installing in routing table
- Throughput replaces bandwidth, Latency replaces delay

### DUAL Terminology
- **Feasible Distance (FD)**: Best metric from this router to destination
- **Reported Distance (RD)** / Advertised Distance (AD): Metric reported by neighbor
- **Successor**: Best next-hop (lowest FD)
- **Feasible Successor (FS)**: Backup path where RD < current FD (feasibility condition)
- **Stuck-in-Active (SIA)**: Query not answered within 3 minutes (default)

### Neighbor Requirements
Must match: K-values, AS number, authentication. Must be on common subnet.

### Cisco IOS EIGRP Named Mode Config
```
router eigrp MYNET
 address-family ipv4 unicast autonomous-system 100
  network 10.0.0.0 0.0.255.255
  af-interface default
   passive-interface
  exit-af-interface
  af-interface GigabitEthernet0/0
   no passive-interface
  exit-af-interface
  topology base
   redistribute static
  exit-af-topology
 exit-address-family
```

---

## IS-IS (Intermediate System to Intermediate System)

### Fundamentals
- **Type**: Link-state, IGP
- **Transport**: Runs directly on Layer 2 (CLNS); NO IP header
- **AD**: 115
- **Metric**: Default 10 per interface (narrow metrics: 6-bit, max 63 per link, 1023 path)
- **Wide metrics**: 24-bit per link (recommended; use `metric-style wide`)

### IS-IS Levels
- **Level 1 (L1)**: Intra-area routing (like OSPF intra-area)
- **Level 2 (L2)**: Inter-area routing (like OSPF backbone)
- **L1/L2 router**: Connects L1 area to L2 backbone (like ABR in OSPF)
- No area 0 concept; L2 forms the backbone

### NET (Network Entity Title)
Format: `49.0001.1921.6800.1001.00`
- Area ID: `49.0001`
- System ID: `1921.6800.1001` (often derived from loopback IP)
- NSEL: `00` (always 00 for routers)

### TLV Types (commonly referenced)
| TLV | Code | Purpose |
|-----|------|---------|
| IS Reachability | 2 | Narrow metric IS neighbors |
| IP Internal Reachability | 128 | Narrow metric IP prefixes |
| Extended IS Reachability | 22 | Wide metric IS neighbors + TE |
| Extended IP Reachability | 135 | Wide metric IP prefixes |
| IPv6 Reachability | 236 | IPv6 prefixes |
| Router Capability | 242 | SR, flex-algo |
| Prefix-SID | — | Sub-TLV of 135/236 for SR |

### IS-IS vs OSPF Design Considerations
- IS-IS preferred in SP/large-scale networks (simpler scaling, no area 0 requirement)
- IS-IS runs on L2 (survives IP misconfig); OSPF requires IP connectivity
- IS-IS supports MPLS/SR TLV extensions natively
- OSPF more common in enterprise (more widely known, richer stub area types)

---

## Segment Routing (SR)

### SR-MPLS
- Labels allocated from a global block (SRGB, default 16000-23999 on IOS-XR)
- Prefix-SID: globally unique, identifies a prefix (absolute or index into SRGB)
- Adjacency-SID: locally significant, identifies a specific link
- Node-SID: Prefix-SID of a loopback (most common)
- Uses IGP extensions (IS-IS TLV 135 sub-TLV, OSPF Opaque LSA) to distribute SIDs

### SRv6
- Uses IPv6 header extension (SRH — Segment Routing Header)
- SIDs are IPv6 addresses (128-bit)
- Functions encoded in SID: End, End.X, End.DT4, End.DT6, End.DX2, etc.
- No MPLS labels; forwarding plane is native IPv6
- Micro-SID (uSID) compresses SIDs for efficiency

### SR-TE (Traffic Engineering)
- Headend computes path; encodes as segment list (label stack or SRH)
- No per-flow state in transit nodes (unlike RSVP-TE)
- SR-PCE (Path Computation Element) provides centralized computation
- On-Demand Nexthop (ODN): auto-create SR-TE policies based on service requirements
- Flex-Algo: define custom algorithms (metric types, constraints) distributed via IGP

### SR vs RSVP-TE
| Aspect | RSVP-TE | SR-TE |
|--------|---------|-------|
| State in transit | Per-LSP state | Stateless core |
| Bandwidth reservation | Native (RSVP signaling) | External (PCE/controller) |
| Scalability | Limited by state | Excellent |
| Automation | Complex | SDN-friendly |
| FRR | Facility backup (bypass) | TI-LFA (topology-independent) |
| Migration | Legacy; widely deployed | Modern; recommended for new designs |
