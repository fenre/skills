---
name: computer-networking
description: >
  CCIE-level computer networking reference covering routing protocols (BGP, OSPF, EIGRP, IS-IS),
  switching (VLANs, STP, VXLAN/EVPN), WAN (MPLS, SD-WAN, DMVPN, IPsec, SASE), wireless (WiFi 5/6/6E/7),
  QoS, IPv6, data center design (spine-leaf), cabling/physical layer, network security, and management
  platforms (Catalyst Center, Meraki, ThousandEyes, Aruba Central, Juniper Mist, Arista CloudVision).
  Multi-vendor CLI coverage for Cisco IOS/IOS-XE/NX-OS/IOS-XR, Juniper Junos, Aruba AOS-CX, Arista EOS,
  and Cato SASE. Use when answering ANY networking question — routing, switching, wireless, cabling,
  protocols, ports, vendor CLI syntax, network design, or troubleshooting.
---

# Computer Networking Reference (CCIE-Level)

## BGP Path Selection Algorithm (Cisco)

Exact order — memorize this; getting it wrong is the most common networking error:

| Step | Attribute | Prefer | Default | Scope |
|------|-----------|--------|---------|-------|
| 0 | Next-hop reachable | Must be valid | — | Mandatory |
| 1 | Weight | Highest | 0 (32768 for local) | Local to router |
| 2 | LOCAL_PREF | Highest | 100 | Within AS |
| 3 | Locally originated | network/aggregate/redistribute over learned | — | Local |
| 4 | AS_PATH | Shortest | — | Global |
| 5 | ORIGIN | IGP < EGP < Incomplete (i < e < ?) | — | Global |
| 6 | MED | Lowest (compared only within same neighbor AS by default) | 0 | Between AS |
| 7 | eBGP over iBGP | eBGP preferred | — | Local |
| 8 | Lowest IGP metric to next-hop | Closest exit (hot-potato) | — | Local |
| 9 | Oldest eBGP path | Most stable | — | Local |
| 10 | Lowest Router ID | Tiebreaker | — | Global |
| 11 | Shortest cluster-list | RR loop avoidance | — | Local |
| 12 | Lowest neighbor address | Final tiebreaker | — | Local |

## OSPF LSA Types

| Type | Name | Originated By | Flooded Within | Purpose |
|------|------|---------------|----------------|---------|
| 1 | Router LSA | Every router | Area | Links and costs within area |
| 2 | Network LSA | DR | Area | Multi-access network info |
| 3 | Summary LSA | ABR | Other areas | Inter-area prefixes |
| 4 | ASBR Summary | ABR | Other areas | Path to ASBR |
| 5 | External LSA | ASBR | Entire domain (not stubs) | External routes |
| 7 | NSSA External | ASBR in NSSA | NSSA only (converted to 5 at ABR) | External routes in NSSA |

OSPF Timers: Hello 10s (broadcast/P2P), 30s (NBMA); Dead = 4x Hello. SPF throttle defaults vary by platform.

## OSPF Area Types

| Area Type | Allows LSA 3 | Allows LSA 5 | Allows LSA 7 | Default Route |
|-----------|-------------|-------------|-------------|---------------|
| Normal | Yes | Yes | No | No (unless configured) |
| Stub | Yes | No | No | Yes (injected by ABR) |
| Totally Stubby | No (only default) | No | No | Yes |
| NSSA | Yes | No | Yes | No (configurable) |
| Totally NSSA | No (only default) | No | Yes | Yes |

## STP Variant Comparison

| Feature | STP (802.1D) | RSTP (802.1w) | PVST+ | Rapid PVST+ | MST (802.1s) |
|---------|-------------|---------------|-------|-------------|--------------|
| Instances | 1 | 1 | 1 per VLAN | 1 per VLAN | Maps VLANs to instances |
| Convergence | 30-50s | 1-6s | 30-50s | 1-6s | 1-6s |
| Standard | IEEE | IEEE | Cisco | Cisco | IEEE |
| Port states | 5 | 3 | 5 | 3 | 3 |
| Default on Cisco | No | No | Legacy | IOS-XE default | No |

STP Timers: Hello 2s, Forward Delay 15s, Max Age 20s.

RSTP port states: Discarding, Learning, Forwarding. RSTP port roles: Root, Designated, Alternate, Backup.

## DSCP Quick Reference

| PHB | DSCP Name | Decimal | Binary | Use Case |
|-----|-----------|---------|--------|----------|
| EF | EF | 46 | 101110 | Voice bearer (priority queue) |
| CS6 | CS6 | 48 | 110000 | Network control (routing protocols) |
| CS5 | CS5 | 40 | 101000 | Signaling / Video signaling |
| AF41 | AF41 | 34 | 100010 | Video conferencing |
| AF31 | AF31 | 26 | 011010 | Mission-critical data |
| AF21 | AF21 | 18 | 010010 | Transactional data |
| AF11 | AF11 | 10 | 001010 | Bulk data |
| CS2 | CS2 | 16 | 010000 | OAM |
| CS1 | CS1 | 8 | 001000 | Scavenger |
| DF/BE | CS0 | 0 | 000000 | Best effort (default) |

AF formula: DSCP = 8*Class + 2*Drop. AF classes 1-4, drop precedence 1-3 (low/med/high).

## Essential Protocol/Port Numbers

| Protocol | Port/Number | Transport | Notes |
|----------|------------|-----------|-------|
| BGP | TCP 179 | TCP | Both peers connect to 179 |
| OSPF | IP Protocol 89 | — | Not TCP/UDP; directly over IP |
| EIGRP | IP Protocol 88 | — | Not TCP/UDP; directly over IP |
| IS-IS | — | — | Runs directly over L2 (no IP header) |
| VXLAN | UDP 4789 | UDP | 24-bit VNI = 16M+ segments |
| SNMP | UDP 161/162 | UDP | 161=queries, 162=traps |
| SNMP Trap | UDP 162 | UDP | |
| SSH | TCP 22 | TCP | |
| Telnet | TCP 23 | TCP | |
| HTTP | TCP 80 | TCP | |
| HTTPS | TCP 443 | TCP | |
| DNS | TCP/UDP 53 | Both | UDP for queries, TCP for zone xfer |
| DHCP | UDP 67/68 | UDP | 67=server, 68=client |
| DHCPv6 | UDP 546/547 | UDP | 546=client, 547=server |
| NTP | UDP 123 | UDP | |
| RADIUS | UDP 1812/1813 | UDP | 1812=auth, 1813=acct |
| TACACS+ | TCP 49 | TCP | Encrypts entire payload |
| Syslog | UDP 514 | UDP | |
| TFTP | UDP 69 | UDP | |
| FTP | TCP 20/21 | TCP | 20=data, 21=control |
| SMTP | TCP 25 | TCP | |
| IMAP | TCP 143 | TCP | |
| LDAP | TCP 389 | TCP | |
| LDAPS | TCP 636 | TCP | |
| NetFlow | UDP 2055/9995/9996 | UDP | Varies by implementation |
| sFlow | UDP 6343 | UDP | |
| IPFIX | UDP 4739 | UDP | |
| LLDP | Ethertype 0x88CC | L2 | IEEE; multicast 01:80:C2:00:00:0E |
| CDP | — | L2 | Cisco proprietary; SNAP |
| LACP | Ethertype 0x8809 | L2 | Slow protocol; multicast 01:80:C2:00:00:02 |
| BFD | UDP 3784/3785 | UDP | 3784=single-hop, 3785=multi-hop |
| VRRP | IP Protocol 112 | — | Multicast 224.0.0.18 |
| HSRP | UDP 1985 (v1) / UDP 2029 (v2) | UDP | HSRPv2 uses 224.0.0.102 |
| GLBP | UDP 3222 | UDP | Cisco proprietary |
| GRE | IP Protocol 47 | — | |
| IPsec ESP | IP Protocol 50 | — | |
| IPsec AH | IP Protocol 51 | — | |
| IKEv2 | UDP 500/4500 | UDP | 4500 for NAT-T |
| MPLS | Ethertype 0x8847/0x8848 | L2 | 0x8847=unicast, 0x8848=multicast |
| LISP | UDP 4342 | UDP | |
| WireGuard | UDP 51820 | UDP | |

## Cabling Quick Reference

| Category | Max Speed | Max Distance | Frequency | Use Case |
|----------|----------|-------------|-----------|----------|
| Cat5e | 1 Gbps | 100m | 100 MHz | Legacy; avoid for new installs |
| Cat6 | 10 Gbps | 55m (37m for 10G) | 250 MHz | SMB / short 10G runs |
| Cat6a | 10 Gbps | 100m | 500 MHz | Enterprise standard; WiFi 6E/7 |
| Cat8 | 25/40 Gbps | 30m | 2000 MHz | Data center short runs only |

| Fiber Type | Core | Max Speed | Typical Distance | Use Case |
|-----------|------|----------|-----------------|----------|
| OM1 | 62.5um MM | 1G | 275m (1G) | Legacy; do not specify new |
| OM2 | 50um MM | 1G | 550m (1G) | Legacy |
| OM3 | 50um MM | 10G/40G/100G | 300m (10G) | Data center |
| OM4 | 50um MM | 10G/40G/100G | 400m (10G), 150m (40/100G) | Data center standard |
| OM5 | 50um MM | 100G/400G | 150m (100G SWDM) | WDM multimode |
| OS1 | 9um SM | 100G+ | 10km | Campus/metro |
| OS2 | 9um SM | 100G+ | 40-80km+ | Long-haul, DCI |

## WiFi Standards Quick Reference

| Standard | Name | Bands | Max Rate | Max Width | Modulation | Key Feature |
|----------|------|-------|----------|-----------|------------|-------------|
| 802.11ac | WiFi 5 | 5 GHz | 3.5 Gbps | 160 MHz | 256-QAM | MU-MIMO DL |
| 802.11ax | WiFi 6 | 2.4/5 GHz | 9.6 Gbps | 160 MHz | 1024-QAM | OFDMA, BSS Color |
| 802.11ax | WiFi 6E | 2.4/5/6 GHz | 9.6 Gbps | 160 MHz | 1024-QAM | 6 GHz band (1200 MHz) |
| 802.11be | WiFi 7 | 2.4/5/6 GHz | 46 Gbps | 320 MHz | 4096-QAM | MLO, 320 MHz channels |

Non-overlapping channels: 2.4 GHz = 1, 6, 11 (20 MHz). 5 GHz = up to 25 (20 MHz). 6 GHz = 59 channels.

## Vendor CLI Rosetta Stone (Essential Commands)

| Task | Cisco IOS/IOS-XE | Juniper Junos | Arista EOS | Aruba AOS-CX |
|------|-----------------|---------------|------------|--------------|
| Show version | `show version` | `show version` | `show version` | `show version` |
| Show routes | `show ip route` | `show route` | `show ip route` | `show ip route` |
| Show interfaces | `show ip int brief` | `show interfaces terse` | `show ip int brief` | `show interface brief` |
| Show BGP | `show ip bgp summary` | `show bgp summary` | `show ip bgp summary` | `show ip bgp summary` |
| Show OSPF neighbors | `show ip ospf neighbor` | `show ospf neighbor` | `show ip ospf neighbor` | `show ip ospf neighbor` |
| Show ARP | `show arp` | `show arp` | `show arp` | `show arp` |
| Show MAC table | `show mac address-table` | `show ethernet-switching table` | `show mac address-table` | `show mac-address-table` |
| Show VLANs | `show vlan brief` | `show vlans` | `show vlan` | `show vlan` |
| Show spanning tree | `show spanning-tree` | `show spanning-tree bridge` | `show spanning-tree` | `show spanning-tree` |
| Show LLDP | `show lldp neighbors` | `show lldp neighbors` | `show lldp neighbors` | `show lldp neighbor-info` |
| Enter config mode | `configure terminal` | `configure` (then `commit`) | `configure terminal` | `configure terminal` |
| Save config | `copy run start` or `write` | `commit` (auto-saved) | `copy run start` or `write` | `copy run start` or `write` |
| Rollback | Archive needed | `rollback N` | `configure replace` | `checkpoint rollback` |

Key difference: Junos uses a candidate config model (changes staged until `commit`). All other vendors apply changes immediately.

Cisco NX-OS difference: uses `feature` command to enable protocols (`feature ospf`, `feature bgp`). No `configure terminal` needed for some contexts.

Cisco IOS-XR difference: uses commit model like Junos (`commit`, `rollback`). Config hierarchy differs from IOS/IOS-XE. Uses `configure`, then `commit`.

## Cisco Platform Operating Systems

| OS | Platforms | Architecture | Config Model | Key Difference |
|----|-----------|-------------|-------------|----------------|
| IOS | Legacy 2900, 3900, ISR G1 | Monolithic | Immediate apply | `copy run start` to persist |
| IOS-XE | Catalyst 9K, ISR 4K, ASR 1K | Linux + IOS daemon | Immediate apply | Same CLI as IOS; runs on Linux |
| NX-OS | Nexus 3K/5K/7K/9K | Linux-based modular | Immediate apply | `feature` to enable protocols |
| IOS-XR | ASR 9K, NCS 5K/5500/540, 8000 | QNX/Linux microkernel | Commit model | `commit` required; `rollback` |

## EIGRP Composite Metric

Default: metric = 256 * (10^7 / min-bandwidth + cumulative-delay/10)

Only K1 (bandwidth) and K3 (delay) are enabled by default (K1=K3=1, K2=K4=K5=0).

Wide metrics (named mode): uses 64-bit values; metric = (K1*BW + K3*Delay + K6*ExtAttr) * 65536/K7.

Feasibility condition: Reported Distance (RD) of successor must be < Feasible Distance (FD) of current best path.

## Additional Resources

For detailed reference on each topic, consult the following files:

- [routing-protocols.md](routing-protocols.md) — BGP, OSPF, EIGRP, IS-IS, Segment Routing deep-dive
- [switching-layer2.md](switching-layer2.md) — VLANs, STP, port-channels, VXLAN/EVPN
- [wan-technologies.md](wan-technologies.md) — MPLS, SD-WAN, DMVPN, IPsec, SASE
- [wireless-networking.md](wireless-networking.md) — WiFi standards, controllers, design
- [network-security.md](network-security.md) — ACLs, NAT, firewalls, 802.1X, first-hop security
- [vendor-cli-reference.md](vendor-cli-reference.md) — Multi-vendor CLI command mapping
- [cabling-physical.md](cabling-physical.md) — Copper, fiber, connectors, PoE, transceivers
- [management-platforms.md](management-platforms.md) — Catalyst Center, Meraki, ThousandEyes, Aruba Central, Mist, CloudVision
- [qos-reference.md](qos-reference.md) — DSCP, queuing, policing, shaping
- [ipv6-reference.md](ipv6-reference.md) — IPv6 addressing, NDP, SLAAC, DHCPv6, transition
- [data-center-design.md](data-center-design.md) — Spine-leaf, VXLAN/EVPN fabric, ACI
- [protocols-ports.md](protocols-ports.md) — Complete protocol/port reference
