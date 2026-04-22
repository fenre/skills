# Multi-Vendor CLI Reference

## Platform Overview

| Vendor | OS | CLI Style | Config Model | Commit/Rollback |
|--------|-----|-----------|-------------|-----------------|
| Cisco | IOS | CLI | Immediate apply | No (copy run start to persist) |
| Cisco | IOS-XE | CLI | Immediate apply | Archive/rollback optional |
| Cisco | NX-OS | CLI | Immediate apply (checkpoint/rollback) | Optional checkpoint |
| Cisco | IOS-XR | CLI | Candidate config | commit required; rollback N |
| Juniper | Junos | CLI | Candidate config | commit required; rollback N |
| Arista | EOS | CLI (IOS-like) | Immediate apply | configure replace; rollback |
| Aruba | AOS-CX | CLI | Immediate apply | checkpoint rollback |

---

## Comprehensive Command Mapping

### System and Management

| Task | IOS / IOS-XE | NX-OS | IOS-XR | Junos | EOS (Arista) | AOS-CX (Aruba) |
|------|-------------|-------|--------|-------|-------------|----------------|
| Show version | `show version` | `show version` | `show version` | `show version` | `show version` | `show version` |
| Show inventory | `show inventory` | `show inventory` | `show inventory` | `show chassis hardware` | `show inventory` | `show system` |
| Show running config | `show running-config` | `show running-config` | `show running-config` | `show configuration` | `show running-config` | `show running-config` |
| Show startup config | `show startup-config` | `show startup-config` | — (committed config) | — (committed) | `show startup-config` | `show startup-config` |
| Save config | `copy run start` / `write` | `copy run start` | `commit` | `commit` | `copy run start` / `write` | `copy run start` / `write` |
| Enter config mode | `configure terminal` | `configure terminal` | `configure` | `configure` | `configure terminal` | `configure terminal` |
| Exit config mode | `end` / `exit` | `end` / `exit` | `end` / `exit` | `exit` / `top` | `end` / `exit` | `end` / `exit` |
| Show logging | `show logging` | `show logging log` | `show logging` | `show log messages` | `show logging` | `show logging` |
| Show CPU | `show processes cpu` | `show processes cpu` | `show processes cpu` | `show chassis routing-engine` | `show processes top` | `show system resource-utilization` |
| Show memory | `show processes memory` | `show processes memory` | `show memory summary` | `show chassis routing-engine` | `show processes top` | `show system resource-utilization` |
| Show users | `show users` | `show users` | `show users` | `show system users` | `show users` | `show users` |
| Reload | `reload` | `reload` | `reload` / `hw-module location all reload` | `request system reboot` | `reload` | `boot system` |
| Set hostname | `hostname NAME` | `hostname NAME` | `hostname NAME` | `set system host-name NAME` | `hostname NAME` | `hostname NAME` |
| Set DNS | `ip name-server X.X.X.X` | `ip name-server X.X.X.X` | `domain name-server X.X.X.X` | `set system name-server X.X.X.X` | `ip name-server X.X.X.X` | `ip dns server-address X.X.X.X` |
| Set NTP | `ntp server X.X.X.X` | `ntp server X.X.X.X` | `ntp server X.X.X.X` | `set system ntp server X.X.X.X` | `ntp server X.X.X.X` | `ntp server X.X.X.X` |
| Set banner | `banner motd #text#` | `banner motd #text#` | `banner login "text"` | `set system login message "text"` | `banner motd "text"` | `banner motd "text"` |

### Interface Configuration

| Task | IOS / IOS-XE | NX-OS | IOS-XR | Junos | EOS (Arista) | AOS-CX (Aruba) |
|------|-------------|-------|--------|-------|-------------|----------------|
| Show interfaces brief | `show ip int brief` | `show ip int brief` | `show ipv4 int brief` | `show interfaces terse` | `show ip int brief` | `show interface brief` |
| Show interface detail | `show int Gi0/0` | `show int Eth1/1` | `show int Gi0/0/0/0` | `show int ge-0/0/0 extensive` | `show int Eth1` | `show int 1/1/1` |
| Shutdown interface | `shutdown` | `shutdown` | `shutdown` | `set int ge-0/0/0 disable` | `shutdown` | `shutdown` |
| Enable interface | `no shutdown` | `no shutdown` | `no shutdown` | `delete int ge-0/0/0 disable` | `no shutdown` | `no shutdown` |
| Set IP address | `ip address X.X.X.X M.M.M.M` | `ip address X.X.X.X/NN` | `ipv4 address X.X.X.X/NN` | `set int ge-0/0/0 unit 0 family inet address X.X.X.X/NN` | `ip address X.X.X.X/NN` | `ip address X.X.X.X/NN` |
| Set description | `description TEXT` | `description TEXT` | `description TEXT` | `set int ge-0/0/0 description "TEXT"` | `description TEXT` | `description TEXT` |
| Set MTU | `mtu 9000` or `ip mtu 9000` | `mtu 9216` | `mtu 9216` | `set int ge-0/0/0 mtu 9216` | `mtu 9214` | `mtu 9198` |
| Set speed/duplex | `speed 1000` / `duplex full` | `speed 1000` / `duplex full` | — (auto-negotiation) | `set int ge-0/0/0 speed 1g` | `speed forced 1000full` | `speed 1000-full` |
| Show counters | `show int counters` | `show int counters` | `show int accounting` | `show int ge-0/0/0 statistics` | `show int counters` | `show int 1/1/1 statistics` |

### VLAN and L2

| Task | IOS / IOS-XE | NX-OS | Junos (ELS) | EOS (Arista) | AOS-CX (Aruba) |
|------|-------------|-------|-------------|-------------|----------------|
| Create VLAN | `vlan 100` / `name DATA` | `vlan 100` / `name DATA` | `set vlans DATA vlan-id 100` | `vlan 100` / `name DATA` | `vlan 100` / `name DATA` |
| Show VLANs | `show vlan brief` | `show vlan` | `show vlans` | `show vlan` | `show vlan` |
| Access port | `switchport mode access` / `switchport access vlan 100` | Same | `set int ge-0/0/0 unit 0 family ethernet-switching interface-mode access vlan members DATA` | `switchport mode access` / `switchport access vlan 100` | `vlan access 100` |
| Trunk port | `switchport mode trunk` / `switchport trunk allowed vlan 100,200` | Same | `set int ge-0/0/0 unit 0 family ethernet-switching interface-mode trunk vlan members [DATA VOICE]` | `switchport mode trunk` / `switchport trunk allowed vlan 100,200` | `vlan trunk allowed 100,200` |
| Native VLAN | `switchport trunk native vlan 999` | Same | `set int ge-0/0/0 native-vlan-id 999` | `switchport trunk native vlan 999` | `vlan trunk native 999` |
| Show MAC table | `show mac address-table` | `show mac address-table` | `show ethernet-switching table` | `show mac address-table` | `show mac-address-table` |
| Show STP | `show spanning-tree` | `show spanning-tree` | `show spanning-tree bridge` | `show spanning-tree` | `show spanning-tree` |

### Routing

| Task | IOS / IOS-XE | NX-OS | IOS-XR | Junos | EOS (Arista) |
|------|-------------|-------|--------|-------|-------------|
| Show routing table | `show ip route` | `show ip route` | `show route` | `show route` | `show ip route` |
| Show BGP summary | `show ip bgp summary` | `show ip bgp summary` | `show bgp summary` | `show bgp summary` | `show ip bgp summary` |
| Show BGP neighbors | `show ip bgp neighbors` | `show ip bgp neighbors` | `show bgp neighbors` | `show bgp neighbor` | `show ip bgp neighbors` |
| Show BGP table | `show ip bgp` | `show ip bgp` | `show bgp` | `show route advertising-protocol bgp X` | `show ip bgp` |
| Show OSPF neighbors | `show ip ospf neighbor` | `show ip ospf neighbors` | `show ospf neighbor` | `show ospf neighbor` | `show ip ospf neighbor` |
| Show OSPF database | `show ip ospf database` | `show ip ospf database` | `show ospf database` | `show ospf database` | `show ip ospf database` |
| Static route | `ip route X.X.X.X M.M.M.M NH` | `ip route X.X.X.X/NN NH` | `router static address-family ipv4 unicast X.X.X.X/NN NH` | `set routing-options static route X.X.X.X/NN next-hop NH` | `ip route X.X.X.X/NN NH` |
| Enable OSPF | `router ospf 1` / `network ... area 0` | `feature ospf` then `router ospf 1` | `router ospf 1` / `area 0 int Lo0` | `set protocols ospf area 0.0.0.0 interface ge-0/0/0` | `router ospf 1` / `network ... area 0` |
| Enable BGP | `router bgp 65001` | `feature bgp` then `router bgp 65001` | `router bgp 65001` | `set protocols bgp group NAME type external peer-as 65002 neighbor X` | `router bgp 65001` |

### Troubleshooting

| Task | IOS / IOS-XE | NX-OS | IOS-XR | Junos | EOS (Arista) |
|------|-------------|-------|--------|-------|-------------|
| Ping | `ping X.X.X.X` | `ping X.X.X.X` | `ping X.X.X.X` | `ping X.X.X.X` | `ping X.X.X.X` |
| Traceroute | `traceroute X.X.X.X` | `traceroute X.X.X.X` | `traceroute X.X.X.X` | `traceroute X.X.X.X` | `traceroute X.X.X.X` |
| Show ARP | `show arp` | `show ip arp` | `show arp` | `show arp` | `show arp` |
| Clear ARP | `clear arp-cache` | `clear ip arp` | `clear arp-cache` | `clear arp` | `clear arp-cache` |
| Show LLDP neighbors | `show lldp neighbors` | `show lldp neighbors` | `show lldp neighbors` | `show lldp neighbors` | `show lldp neighbors` |
| Show CDP neighbors | `show cdp neighbors` | `show cdp neighbors` | — (no CDP) | — (no CDP) | `show lldp neighbors` |
| Debug/monitor | `debug ip ...` | `debug ip ...` | `debug ...` | `monitor traffic interface ...` | `debug ...` |
| Show tech | `show tech-support` | `show tech-support` | `show tech-support` | `request support information` | `show tech-support` |
| Packet capture | `monitor capture ...` | `ethanalyzer local interface ...` | `monitor interface ...` | `monitor traffic ...` | `bash tcpdump -i ...` |

---

## Key Platform-Specific Differences

### NX-OS Specifics
- Must enable features: `feature ospf`, `feature bgp`, `feature vpc`, `feature lacp`, etc.
- Uses `/NN` CIDR notation for IP addresses (not dotted-decimal masks)
- vPC (Virtual Port-Channel) for MLAG: requires peer-link + peer-keepalive
- `show running-config diff` to see changes
- Supports checkpoint/rollback: `checkpoint`, `rollback running-config checkpoint NAME`

### IOS-XR Specifics
- Candidate configuration model (like Junos): `commit` required
- `show configuration changes` to see uncommitted changes
- `rollback configuration last N` to revert
- No `configure terminal`; uses `configure` or `configure exclusive`
- Process restartability: individual processes can restart without full reload
- Administration plane separate: `admin` mode for hardware management
- Bundle interfaces: `interface Bundle-Ether1` (not Port-channel)
- Route-policy replaces route-map: `route-policy NAME ... end-policy`

### Junos Specifics
- Hierarchical configuration; navigate with `edit`, `set`, `delete`
- `show | compare` to see uncommitted changes
- `commit check` validates syntax before applying
- `commit confirmed N` auto-rollback after N minutes if not re-confirmed
- `rollback N` where N=0 is current committed, N=1 is previous, etc. (keeps 50 rollbacks)
- No `no` prefix; use `delete` to remove config
- Operational mode (`>`) vs Configuration mode (`#`)
- `show | display set` to see config as set commands
- `request system snapshot` for backup
- Groups and apply-groups for config templates

### Arista EOS Specifics
- Very IOS-like CLI (easiest transition from Cisco)
- Native Linux: `bash` command enters Linux shell
- `daemon` for custom agents
- Uses CIDR notation like NX-OS for most commands
- MLAG: `mlag configuration` with peer-link and peer-address
- CloudVision integration via TerminAttr agent streaming to CVP
- eAPI: RESTful API; `management api http-commands` to enable
- `configure session NAME` for candidate config (similar to commit model)

### Aruba AOS-CX Specifics
- REST API built-in; every CLI command has API equivalent
- `checkpoint` for config versioning; `checkpoint rollback NAME`
- VSX (Virtual Switching Extension): MLAG equivalent
- VSF (Virtual Switching Framework): switch stacking
- NAE (Network Analytics Engine): on-box Python agents for monitoring
- `show running-config interface 1/1/1` works like Cisco
- Interface naming: `1/1/1` (member/slot/port)
