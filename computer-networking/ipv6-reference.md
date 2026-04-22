# IPv6 Deep Reference

## IPv6 Address Format
- 128-bit address; 8 groups of 4 hex digits separated by colons
- Example: `2001:0db8:0000:0000:0000:0000:0000:0001`
- Shorthand: omit leading zeros, replace longest consecutive all-zero groups with `::` (once only)
- Shortened: `2001:db8::1`

## IPv6 Address Types

### Unicast Addresses

| Type | Prefix | Scope | Purpose |
|------|--------|-------|---------|
| Global Unicast (GUA) | `2000::/3` (currently `2001::/16` + `2400-2c00::/12`) | Global | Equivalent of public IPv4; routable on internet |
| Link-Local | `fe80::/10` | Link only | Auto-configured on every interface; NOT routable |
| Unique Local (ULA) | `fc00::/7` (practically `fd00::/8`) | Organization | Equivalent of RFC 1918; NOT routable on internet |
| Loopback | `::1/128` | Host | Equivalent of 127.0.0.1 |
| Unspecified | `::/128` | N/A | Equivalent of 0.0.0.0; used as source before address assignment |

### Multicast Addresses

| Address | Scope | Purpose |
|---------|-------|---------|
| `ff02::1` | Link-local | All nodes (like broadcast) |
| `ff02::2` | Link-local | All routers |
| `ff02::5` | Link-local | All OSPF routers |
| `ff02::6` | Link-local | All OSPF DRs |
| `ff02::9` | Link-local | All RIPng routers |
| `ff02::a` | Link-local | All EIGRP routers |
| `ff02::d` | Link-local | All PIM routers |
| `ff02::1:2` | Link-local | All DHCPv6 relay agents and servers |
| `ff05::1:3` | Site-local | All DHCPv6 servers |
| `ff02::1:ffXX:XXXX` | Link-local | Solicited-node multicast (NDP) |

### Anycast
- Same address assigned to multiple interfaces (typically routers)
- Packet delivered to nearest (by routing) instance
- Used for DNS root servers, default gateways, VXLAN anycast VTEP

### Solicited-Node Multicast
- Formed from last 24 bits of unicast/anycast address
- Prefix: `ff02::1:ff00:0/104`
- Example: Address `2001:db8::1234:5678` → Solicited-node `ff02::1:ff34:5678`
- Used by NDP for efficient neighbor discovery (no broadcast)

---

## IPv6 Header
- **Fixed header**: 40 bytes (vs IPv4's 20-60 bytes)
- **Fields**: Version (4), Traffic Class (8), Flow Label (20), Payload Length (16), Next Header (8), Hop Limit (8), Source (128), Destination (128)
- **No checksum** in IPv6 header (handled by upper layers)
- **No fragmentation** by routers; source must use Path MTU Discovery
- **Extension headers**: chained via Next Header field (Hop-by-Hop, Routing, Fragment, Destination, AH, ESP, Mobility)
- **Minimum MTU**: 1280 bytes (recommended 1500)

---

## NDP (Neighbor Discovery Protocol — RFC 4861)

Replaces ARP, ICMP Router Discovery, and ICMP Redirect in IPv4.

### NDP Message Types (ICMPv6)

| Type | Code | Name | Purpose |
|------|------|------|---------|
| 133 | 0 | Router Solicitation (RS) | Host asks for router |
| 134 | 0 | Router Advertisement (RA) | Router announces presence, prefix, flags |
| 135 | 0 | Neighbor Solicitation (NS) | Address resolution (like ARP request) + DAD |
| 136 | 0 | Neighbor Advertisement (NA) | Response to NS (like ARP reply) |
| 137 | 0 | Redirect | Router tells host about better next-hop |

### NDP Functions
1. **Router Discovery**: Hosts send RS; routers respond with RA
2. **Prefix Discovery**: RA contains on-link prefixes for SLAAC
3. **Address Resolution**: NS/NA replace ARP
4. **Duplicate Address Detection (DAD)**: NS sent to solicited-node multicast before using address
5. **Neighbor Unreachability Detection (NUD)**: Periodic NS probes to verify reachability
6. **Redirect**: Inform hosts of better path

### RA Flags (critical for SLAAC/DHCPv6 decision)
| Flag | Name | Meaning |
|------|------|---------|
| A | Autonomous | Prefix can be used for SLAAC (1 = yes) |
| L | On-Link | Prefix is on this link (1 = yes) |
| M | Managed | Use DHCPv6 for addresses (1 = use DHCPv6 stateful) |
| O | Other | Use DHCPv6 for other info (DNS, NTP) (1 = use DHCPv6 stateless) |

**Common combinations:**
- A=1, M=0, O=0: SLAAC only (no DHCPv6)
- A=1, M=0, O=1: SLAAC for address + DHCPv6 for DNS/options
- A=0, M=1, O=1: DHCPv6 stateful (full DHCPv6 for address + options)
- A=1, M=1, O=1: Both SLAAC and DHCPv6 (client gets address from both; implementation-dependent)

---

## Address Assignment Methods

### SLAAC (Stateless Address Autoconfiguration — RFC 4862)
1. Interface generates link-local address (fe80:: + EUI-64 or random)
2. DAD verifies uniqueness
3. Host sends RS → Router replies with RA containing prefix + A flag
4. Host creates GUA: prefix + interface ID (EUI-64 or RFC 7217 stable privacy)
5. No central record of assigned addresses (stateless)

### EUI-64 Interface ID
- Insert `ff:fe` in middle of 48-bit MAC address; flip 7th bit (U/L bit)
- Example: MAC `00:1a:2b:3c:4d:5e` → EUI-64 `021a:2bff:fe3c:4d5e`
- Privacy concern: reveals MAC address → prefer RFC 7217 or temporary addresses (RFC 8981)

### DHCPv6 Stateful (RFC 8415)
- Server assigns address + options (DNS, domain, NTP)
- Uses UDP 546 (client) and UDP 547 (server)
- 4-message exchange: Solicit → Advertise → Request → Reply
- Rapid commit (2-message): Solicit → Reply
- Prefix Delegation (PD): DHCPv6-PD assigns /48 or /56 to requesting router
- DUID (DHCP Unique Identifier) identifies clients (not MAC like DHCPv4)

### DHCPv6 Stateless
- Server provides options only (DNS, NTP); NOT addresses
- Address from SLAAC; options from DHCPv6
- 2-message: Information-Request → Reply

### Comparison

| Feature | SLAAC | DHCPv6 Stateless | DHCPv6 Stateful |
|---------|-------|-----------------|-----------------|
| Address assignment | Yes (from RA prefix) | No | Yes |
| DNS/options | No (needs DHCPv6) | Yes | Yes |
| Central logging | No | No | Yes |
| Prefix Delegation | No | No | Yes |
| RA flags needed | A=1 | A=1, O=1 | M=1 |

---

## IPv6 Subnetting

### Recommended Allocations (RFC 6177)
| Assignment | Prefix Length | Addresses |
|-----------|-------------|-----------|
| RIR → ISP | /32 | 2^96 |
| ISP → Customer (minimum) | /48 | 2^80 (65,536 /64 subnets) |
| ISP → Customer (residential) | /56 | 2^72 (256 /64 subnets) |
| Subnet (LAN) | /64 | 2^64 (required for SLAAC) |
| Point-to-point link | /64 or /127 | /127 recommended (RFC 6164) |
| Loopback | /128 | Single address |

- **/64 is mandatory for SLAAC** to work (interface ID = 64 bits)
- Never use anything other than /64 for subnets with hosts unless you have a specific reason
- Point-to-point links: /127 avoids ping-pong issue with /64 and subnet router anycast

---

## IPv6 Transition Mechanisms

### Dual-Stack (Recommended)
- Run both IPv4 and IPv6 simultaneously on all devices
- OS prefers IPv6 when both available (Happy Eyeballs / RFC 8305)
- Most straightforward; requires IPv6 on entire path
- Industry standard approach

### Tunneling
| Mechanism | Type | Use Case | Status |
|-----------|------|----------|--------|
| 6in4 (Manual tunnel) | Static, configured | Connecting IPv6 islands over IPv4 | Active |
| GRE | Point-to-point | IPv6 over IPv4 GRE tunnel | Active |
| 6to4 (RFC 3056) | Automatic, 2002::/16 | Deprecated by RFC 7526 | Deprecated |
| 6rd (RFC 5969) | ISP-managed | ISP IPv6 deployment over IPv4 | Active |
| Teredo | NAT traversal | Deprecated | Deprecated |
| ISATAP | Intra-site | Deprecated by RFC 7526 | Deprecated |
| DS-Lite (RFC 6333) | IPv4-in-IPv6 | Carry IPv4 over IPv6-only core | Active (ISP) |
| MAP-E / MAP-T | Stateless | ISP-scale IPv4 over IPv6 | Active (ISP) |

### Translation
| Mechanism | Direction | Stateful | Use Case |
|-----------|-----------|----------|----------|
| NAT64 (RFC 6146) | IPv6 → IPv4 | Yes | IPv6-only clients reaching IPv4 servers |
| DNS64 (RFC 6147) | — | — | Synthesizes AAAA from A records for NAT64 |
| SIIT (RFC 7915) | Bidirectional | No | 1:1 algorithmic translation |
| 464XLAT (RFC 6877) | IPv4-in-IPv6-over-NAT64 | Yes | Mobile: IPv4 app on IPv6-only network |
| CLAT | Client-side | — | Part of 464XLAT; translates IPv4→IPv6 on device |
| PLAT | Provider-side | Yes | Part of 464XLAT; NAT64 at provider |

### Well-Known NAT64 Prefix
- `64:ff9b::/96` — Embeds IPv4 address in last 32 bits
- Example: `64:ff9b::192.168.1.1` = `64:ff9b::c0a8:0101`

---

## Cisco IPv6 Configuration

### Basic IPv6 on IOS-XE
```
ipv6 unicast-routing
!
interface GigabitEthernet0/0
 ipv6 address 2001:db8:1::1/64
 ipv6 address fe80::1 link-local
 ipv6 nd prefix 2001:db8:1::/64 14400 14400 no-autoconfig
 ipv6 dhcp server DHCPv6-POOL
 ipv6 nd managed-config-flag
 ipv6 nd other-config-flag
```

### DHCPv6 Server
```
ipv6 dhcp pool DHCPv6-POOL
 address prefix 2001:db8:1::/64
 dns-server 2001:4860:4860::8888
 domain-name example.com
```

### OSPFv3
```
router ospfv3 1
 address-family ipv6 unicast
  router-id 1.1.1.1
 exit-address-family
!
interface GigabitEthernet0/0
 ospfv3 1 ipv6 area 0
```

### IPv6 ACL
```
ipv6 access-list BLOCK-TELNET-V6
 deny tcp any any eq 23
 permit ipv6 any any
!
interface GigabitEthernet0/0
 ipv6 traffic-filter BLOCK-TELNET-V6 in
```
