# Data Center Design Deep Reference

## Architecture Evolution

| Era | Topology | Oversubscription | Traffic Pattern | Technology |
|-----|----------|-----------------|-----------------|------------|
| Traditional | 3-tier (Core/Agg/Access) | 20:1 at core | North-South (client→server) | VLANs, STP |
| Modern | Spine-Leaf (Clos) | 1:1 to 3:1 | East-West (server→server) | VXLAN/EVPN, ECMP |
| Hyper-converged | Spine-Leaf + SDN | 1:1 | East-West dominant | ACI, NSX, fabric |

---

## Spine-Leaf (Clos Fabric) Architecture

### Design Principles
- **Every leaf connects to every spine**: Full-mesh bipartite graph
- **Max 2 hops** between any two servers
- **ECMP everywhere**: Equal-cost multipath across all spine paths
- **Scale-out**: Add leaves for more server ports; add spines for more bandwidth
- **No spanning tree** in the fabric; L3 to the leaf (or VXLAN overlay)

### Physical Design Rules

| Parameter | Recommendation |
|-----------|---------------|
| Leaf-to-spine uplinks | 4-8 links (ECMP) |
| Oversubscription ratio | 1:1 (non-blocking) to 3:1 (acceptable) |
| Leaf ports | 48-port typical; mix of server + uplink |
| Spine count | Minimum 2; typically 4 for production |
| Maximum leaves per spine | Limited by spine port count |
| Leaf-to-server | 10G/25G (access), 100G/400G (GPU/storage) |
| Leaf-to-spine | 40G/100G/400G depending on scale |

### Cabling Guidelines
| Connection | Cable Type | Typical Distance |
|-----------|-----------|-----------------|
| Server to leaf (same rack) | DAC (Direct Attach Copper) | 1-3m |
| Leaf to spine (same row) | AOC or OM4 SR fiber | 5-30m |
| Leaf to spine (cross-row) | OM4 or OS2 fiber | 30-500m |
| DCI (Data Center Interconnect) | OS2 + ZR/ZR+ optics | 10-120+ km |

### Underlay Design
- **Protocol**: eBGP (most common in hyperscale/modern) or OSPF/IS-IS (traditional)
- **eBGP Underlay**: Each leaf = unique ASN; each spine = unique ASN; simple and scalable
- **OSPF Underlay**: Single area (area 0) across fabric; point-to-point links
- **IS-IS Underlay**: Single L2 domain; wide metrics
- Keep underlay simple; avoid redistribution, filters, or complex policies
- Loopbacks: /32 for VTEP source; advertised into underlay IGP/BGP
- BFD: Enable between leaf-spine for sub-second failure detection

### eBGP Underlay Example (Arista EOS)
```
router bgp 65101
   router-id 10.0.0.1
   no bgp default ipv4-unicast
   maximum-paths 4 ecmp 4
   neighbor SPINE peer group
   neighbor SPINE remote-as 65001
   neighbor SPINE bfd
   neighbor 10.1.1.1 peer group SPINE
   neighbor 10.1.2.1 peer group SPINE
   !
   address-family ipv4
      neighbor SPINE activate
      network 10.0.0.1/32
```

---

## VXLAN/EVPN Fabric

### Overlay Design

**Symmetric IRB** (Integrated Routing and Bridging) — Standard model:
- Each leaf can route between VNIs locally (distributed anycast gateway)
- Uses L3 VNI (transit VNI) for inter-subnet traffic
- Anycast gateway: same MAC + IP on all leaves for a given SVI
- Ingress leaf routes, encapsulates in VXLAN with L3 VNI; egress leaf decapsulates and bridges
- Scales: only need VLANs/VNIs that are locally present (not all VNIs everywhere)

**Asymmetric IRB** — Simpler but less scalable:
- Ingress leaf routes between subnets locally; egress leaf only bridges
- Requires ALL VNIs present on ALL leaves (does not scale)
- Avoid for production fabrics

### Anycast Gateway
- All leaves share the same virtual MAC (e.g., `0000.2222.3333`) and same IP gateway per VLAN
- Host ARP/ND responses are always from local leaf → instant gateway, no FHRP needed
- Eliminates HSRP/VRRP/GLBP in the fabric
- Cisco NX-OS: `fabric forwarding anycast-gateway-mac`
- Arista EOS: `ip virtual-router mac-address`

### Multi-Tenancy
- **VRF per tenant**: Each tenant has isolated routing (L3 VNI per VRF)
- **VNI per network segment**: L2 VNI per VLAN per tenant
- Route-targets (RT) control route import/export between VRFs
- Route Distinguisher (RD) ensures uniqueness in BGP table

### Border/Edge Leaf
- Connects fabric to external networks (WAN, internet, other DCs)
- Runs external BGP/OSPF/static toward WAN routers
- Redistributes external routes into EVPN (Type 5 IP Prefix routes)
- May provide firewall service insertion (PBR or service chaining)

---

## Cisco ACI (Application Centric Infrastructure)

### Architecture
| Component | Role |
|-----------|------|
| APIC (Application Policy Infrastructure Controller) | Centralized SDN controller cluster (3+ nodes) |
| Spine | ACI fabric spine switches (Nexus 9500) |
| Leaf | ACI fabric leaf switches (Nexus 9300/9400) |
| Fabric | IS-IS underlay + VXLAN overlay (no manual config) |

### ACI Object Model

```
Tenant
├── VRF (Context)
│   └── Bridge Domain (BD)
│       └── Subnet (gateway)
├── Application Profile (AP)
│   └── EPG (Endpoint Group)
│       ├── Static/Dynamic port binding
│       └── Contracts (consumed/provided)
├── Contract
│   ├── Subject
│   │   └── Filter (L3/L4 rules)
│   └── Scope (VRF, Tenant, Global)
└── L3Out (External routed connectivity)
    └── External EPG
```

### Key ACI Concepts
- **EPG**: Group of endpoints with same policy; NOT a VLAN (though mapped to VLANs)
- **Contract**: Whitelist policy between EPGs (by default ALL traffic denied between EPGs)
- **Bridge Domain**: L2 flooding domain; contains subnets; mapped to VNI
- **VRF/Context**: L3 routing domain; like a VRF
- **Fabric Access Policies**: Physical infrastructure configuration (ports, VLANs, domains, AAEP)

### ACI vs Traditional VXLAN/EVPN
| Aspect | ACI | Standalone VXLAN/EVPN |
|--------|-----|----------------------|
| Controller | Required (APIC) | Optional (standalone leaf config) |
| Policy model | Whitelist (deny by default) | No policy (routing-only) |
| Multi-tenancy | Built-in (Tenant/VRF/EPG) | VRF-based |
| Micro-segmentation | Contract-based | Requires external firewall |
| Configuration | Declarative (API/GUI) | CLI per device |
| Interop | ACI-only leaves | Multi-vendor (NX-OS, EOS, Junos) |
| Learning | Hardware proxy (spine) | BGP control-plane |

### ACI API (APIC REST API)
- RESTful; XML or JSON
- Base URL: `https://<apic>/api/`
- Authentication: `POST /api/aaaLogin.json` with credentials
- Object model: Managed Objects (MOs) with distinguished names (DN)
- Example: `GET /api/node/class/fvTenant.json` → list all tenants

---

## Data Center Interconnect (DCI)

### DCI Technologies
| Technology | Use Case | Distance |
|-----------|----------|----------|
| VXLAN EVPN Multi-Site | L2/L3 extension between fabrics | Metro/regional |
| OTV (Overlay Transport Virtualization) | L2 extension over L3 (Cisco) | Metro |
| VPLS/EVPN over MPLS | SP-based L2 extension | Any |
| Dark Fiber + 400ZR/ZR+ | Direct optical connection | 10-120+ km |
| DWDM | Dense wavelength division; many channels on one fiber | Metro/long-haul |
| SD-WAN | Application-aware DCI | Any |

### EVPN Multi-Site (Cisco NX-OS)
- Extends EVPN across multiple VXLAN fabrics
- Border Gateway (BGW) on each site re-originates routes
- Avoids full mesh of BGP sessions between all leaves across sites
- Per-site autonomy: each fabric is independent with own spines/controllers
- Multicast: ingress-replication preferred; avoids multicast across DCI

### Stretched vs Localized VLANs
- **Best practice**: Avoid stretching L2 across data centers
- If needed: use EVPN multi-site with per-site anycast gateway
- Avoid STP across DCI at all costs
- Use L3 DCI (eBGP between border leaves/routers) when possible

---

## Data Center Network Design Best Practices

### IP Addressing
- Use /31 or /30 for point-to-point links (leaf-spine)
- Use /32 loopbacks for VTEP source and router-ID
- Plan large enough blocks for growth; use summary-friendly addressing
- Separate underlay addressing from overlay addressing

### Failure Domains
- Each leaf pair = failure domain for attached servers
- Each spine = potential bandwidth reduction (not outage) if lost
- Design for N-1 spine failure: fabric still operates at reduced capacity
- Dual-homed servers (MLAG/EVPN ESI) survive single leaf failure

### MTU
- Underlay MTU: 9214 (jumbo) recommended to accommodate VXLAN overhead (50 bytes)
- Overlay effective MTU: underlay MTU - 50 = 9164 for VXLAN-encapsulated traffic
- Verify MTU end-to-end including spine links

### Monitoring and Observability
- Streaming telemetry (gNMI/gRPC) for real-time metrics
- Syslog for event logging
- NetFlow/IPFIX/sFlow for traffic analysis
- SNMP for legacy monitoring
- BFD for sub-second failure detection
- Fabric-wide dashboards: Catalyst Center, CloudVision, or custom (Splunk/Grafana)

### Security in the Data Center
- Micro-segmentation: ACI Contracts, SGTs, or distributed firewall (NSX)
- East-west firewalling: service-chain or distributed (more scalable than hairpinning)
- Encryption: MACsec for leaf-spine links; IPsec for DCI
- Zero Trust: authenticate every workload; least-privilege access
