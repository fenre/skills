# Network Security Deep Reference

## Access Control Lists (ACLs)

### Cisco ACL Types
| Type | Number Range | Matches On | Platform |
|------|-------------|------------|----------|
| Standard (numbered) | 1-99, 1300-1999 | Source IP only | IOS/IOS-XE |
| Extended (numbered) | 100-199, 2000-2699 | Src/Dst IP, protocol, port | IOS/IOS-XE |
| Named (standard) | Name | Source IP | IOS/IOS-XE |
| Named (extended) | Name | Src/Dst IP, protocol, port | IOS/IOS-XE |
| NX-OS ACL | Name | Full match | NX-OS |
| IOS-XR ACL | Name | Full match; uses commit model | IOS-XR |

### ACL Processing Rules
- Processed top-down; first match wins
- Implicit `deny any` at end of every ACL
- Standard ACL: place close to destination
- Extended ACL: place close to source
- One ACL per interface, per direction, per protocol

### Cisco IOS Extended ACL Example
```
ip access-list extended BLOCK-TELNET
 deny tcp any any eq 23 log
 permit ip any any
!
interface GigabitEthernet0/0
 ip access-group BLOCK-TELNET in
```

### Juniper Junos Firewall Filter
```
firewall {
    family inet {
        filter BLOCK-TELNET {
            term DENY-TELNET {
                from {
                    protocol tcp;
                    destination-port 23;
                }
                then {
                    discard;
                    log;
                }
            }
            term ALLOW-REST {
                then accept;
            }
        }
    }
}
interfaces {
    ge-0/0/0 {
        unit 0 {
            family inet {
                filter {
                    input BLOCK-TELNET;
                }
            }
        }
    }
}
```

### Arista EOS ACL
```
ip access-list BLOCK-TELNET
   deny tcp any any eq telnet log
   permit ip any any
!
interface Ethernet1
   ip access-group BLOCK-TELNET in
```

---

## NAT (Network Address Translation)

### NAT Types
| Type | Cisco Term | Description | Use Case |
|------|-----------|-------------|----------|
| Static NAT | `ip nat inside source static` | 1:1 mapping | Servers needing fixed public IP |
| Dynamic NAT | `ip nat inside source list ... pool` | Many:Many from pool | Pool of public IPs |
| PAT/NAT Overload | `ip nat inside source list ... overload` | Many:1 using ports | Most common; single public IP |
| Policy NAT | Route-map based | Match on src+dst | VPN scenarios |
| Twice NAT | Source + Destination | Bidirectional translation | Overlapping subnets |

### NAT Order of Operations (Cisco)
- **Inside-to-Outside**: Route lookup → NAT (inside→outside) → Forward
- **Outside-to-Inside**: NAT (outside→inside) → Route lookup → Forward
- ACLs applied BEFORE NAT on inbound, AFTER NAT on outbound

### Cisco NAT Config Example
```
ip nat inside source list NAT-ACL interface GigabitEthernet0/1 overload
!
ip access-list standard NAT-ACL
 permit 10.0.0.0 0.0.255.255
!
interface GigabitEthernet0/0
 ip nat inside
interface GigabitEthernet0/1
 ip nat outside
```

### NAT64
- Translates between IPv6-only clients and IPv4-only servers
- **Stateful NAT64**: Many-to-one; maintains session state (most common)
- **Stateless NAT64 (SIIT)**: 1:1 algorithmic translation
- **DNS64**: Synthesizes AAAA records from A records using well-known prefix 64:ff9b::/96
- Used in IPv6-only deployments to reach legacy IPv4 services

---

## 802.1X (Port-Based Network Access Control)

### Components
- **Supplicant**: Client device (laptop, phone, IoT)
- **Authenticator**: Switch or WLC (NAS)
- **Authentication Server**: RADIUS server (ISE, FreeRADIUS, NPS)

### EAP Methods
| Method | Credentials | Mutual Auth | Certificate Required | Use Case |
|--------|------------|-------------|---------------------|----------|
| EAP-TLS | Certificate | Yes | Both sides | Most secure; enterprise |
| PEAP (MSCHAPv2) | Username/password | Server cert only | Server only | Most common enterprise |
| EAP-TTLS | Username/password | Server cert only | Server only | Alternative to PEAP |
| EAP-FAST | PAC or cert | Yes | Optional | Cisco; fast re-auth |
| MAB | MAC address | No | No | Printers, IoT fallback |
| EAP-TEAP | Multiple | Yes | Server | RFC 7170; successor to PEAP+EAP-FAST |

### 802.1X Port States
- **Unauthorized**: Only EAP/802.1X traffic allowed
- **Authorized**: Full network access after successful authentication
- **Force-authorized**: Always authorized (default on many switches)
- **Force-unauthorized**: Always blocked

### 802.1X Host Modes
| Mode | Behavior |
|------|----------|
| Single-host | One MAC authenticated; all others blocked |
| Multi-host | One MAC authenticates; all others piggyback |
| Multi-domain | One data device + one voice device |
| Multi-auth | Each MAC authenticated independently |

### Cisco 802.1X Config
```
aaa new-model
aaa authentication dot1x default group radius
aaa authorization network default group radius
dot1x system-auth-control
!
radius server ISE-1
 address ipv4 10.1.1.100 auth-port 1812 acct-port 1813
 key RADIUS-SECRET
!
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 100
 authentication port-control auto
 authentication host-mode multi-auth
 dot1x pae authenticator
 mab
 authentication order dot1x mab
 authentication priority dot1x mab
 spanning-tree portfast
```

---

## MACsec (IEEE 802.1AE)

### Overview
- Layer 2 hop-by-hop encryption (not end-to-end)
- Encrypts Ethernet frames between two directly connected devices
- Uses GCM-AES-128 or GCM-AES-256
- **MKA (MACsec Key Agreement)**: IEEE 802.1X-2010; negotiates encryption keys

### MACsec Modes
| Mode | Encryption | Authentication | Use Case |
|------|-----------|---------------|----------|
| should-secure | Encrypt if peer supports | Fallback to clear | Migration |
| must-secure | Always encrypt | Drop if no MACsec | Production security |

### Deployment Models
- **Switch-to-switch (Downlink MACsec)**: Between infrastructure devices
- **Switch-to-host (Uplink MACsec)**: Between switch and endpoint (requires client support)
- **WAN MACsec**: Over dark fiber or DWDM (requires hardware support)

---

## First-Hop Security (FHS)

### DHCP Snooping
- Builds binding table: MAC ↔ IP ↔ VLAN ↔ Port
- **Trusted ports**: Uplinks to DHCP server
- **Untrusted ports**: Access ports; only DHCP requests allowed
- Drops rogue DHCP server offers on untrusted ports
- Required foundation for DAI and IP Source Guard

```
ip dhcp snooping
ip dhcp snooping vlan 100,200
!
interface GigabitEthernet1/0/24
 ip dhcp snooping trust
```

### Dynamic ARP Inspection (DAI)
- Validates ARP packets against DHCP snooping binding table
- Drops ARP with mismatched MAC/IP on untrusted ports
- Prevents ARP spoofing/poisoning attacks

```
ip arp inspection vlan 100,200
!
interface GigabitEthernet1/0/24
 ip arp inspection trust
```

### IP Source Guard (IPSG)
- Filters IP traffic based on DHCP snooping binding table
- Drops traffic from IPs not in binding table
- Prevents IP spoofing

```
interface GigabitEthernet1/0/1
 ip verify source
```

### IPv6 First-Hop Security
| Feature | Purpose | Cisco Command |
|---------|---------|---------------|
| RA Guard | Block rogue Router Advertisements | `ipv6 nd raguard` |
| DHCPv6 Guard | Block rogue DHCPv6 servers | `ipv6 dhcp guard` |
| ND Inspection | Validate Neighbor Discovery | `ipv6 nd inspection` |
| Source Guard v6 | Validate IPv6 source addresses | `ipv6 source-guard` |
| Destination Guard | Prevent ND cache exhaustion | `ipv6 destination-guard` |

---

## Control Plane Protection

### CoPP (Control Plane Policing)
- Rate-limits traffic destined to the switch/router CPU
- Protects against control plane DoS
- Uses class-maps to classify and policy-maps to rate-limit
- Standard on NX-OS (default CoPP policy); recommended on IOS-XE

### Storm Control
- Limits broadcast, multicast, or unknown unicast traffic per interface
- Threshold: percentage of bandwidth or packets per second
- Action: shutdown, trap, or error-disable

```
interface GigabitEthernet1/0/1
 storm-control broadcast level 10.00
 storm-control multicast level 10.00
 storm-control action shutdown
```

### URPF (Unicast Reverse Path Forwarding)
- Verifies source IP has a return route via the receiving interface
- **Strict mode**: Source must be reachable via receiving interface
- **Loose mode**: Source must exist in routing table (any interface)
- Prevents IP spoofing at network edge

```
interface GigabitEthernet0/0
 ip verify unicast source reachable-via rx
```

---

## Network Segmentation

### Macro-Segmentation
- VLANs + Inter-VLAN routing via L3 switch or firewall
- VRFs for multi-tenant isolation
- Firewall zones (inside/outside/DMZ)

### Micro-Segmentation
- **Cisco TrustSec / SGT**: Security Group Tags assigned at authentication; policy enforced at egress
- **Cisco ACI Contracts**: Whitelist model; EPG-to-EPG policy
- **VMware NSX**: Distributed firewall; per-VM micro-segmentation
- **Zero Trust**: Verify every access request; never trust by default

### TrustSec/SGT Overview
1. User authenticates via 802.1X/MAB → ISE assigns SGT
2. SGT propagated inline (802.1Q CMD header) or via SXP (SGT Exchange Protocol)
3. SGACL enforced at egress switch based on source-SGT → destination-SGT matrix
4. Policy defined centrally in ISE; pushed to network devices

---

## ZTP (Zero-Touch Provisioning)

| Vendor | Method | Config Source | Protocol |
|--------|--------|---------------|----------|
| Cisco IOS-XE | PnP (Plug and Play) | Catalyst Center / PnP server | HTTPS |
| Cisco NX-OS | POAP (PowerOn Auto Provisioning) | DHCP + script server | TFTP/HTTP |
| Cisco IOS-XR | ZTP | DHCP + config server | HTTP/TFTP |
| Arista EOS | ZTP | DHCP option 67 + config | HTTP/TFTP |
| Juniper | ZTP / Phone Home | DHCP + redirect server | HTTPS |
| Aruba AOS-CX | ZTP | Aruba Central / DHCP | HTTPS |

### Cisco PnP Flow
1. Device boots with no config → sends DHCP request
2. DHCP server returns option 43 (PnP server IP) or DNS resolves `pnpserver.domain.com`
3. Device contacts PnP server (Catalyst Center) via HTTPS
4. Catalyst Center pushes day-0 config, image, and assigns to site
