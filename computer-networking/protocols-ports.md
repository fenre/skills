# Protocols & Port Numbers Complete Reference

## IP Protocol Numbers

These are NOT TCP/UDP ports. They identify the protocol in the IP header (Protocol field).

| Number | Protocol | Description |
|--------|----------|-------------|
| 1 | ICMP | Internet Control Message Protocol |
| 2 | IGMP | Internet Group Management Protocol |
| 6 | TCP | Transmission Control Protocol |
| 17 | UDP | User Datagram Protocol |
| 41 | IPv6 encapsulation | IPv6-in-IPv4 tunneling (6in4) |
| 47 | GRE | Generic Routing Encapsulation |
| 50 | ESP | IPsec Encapsulating Security Payload |
| 51 | AH | IPsec Authentication Header |
| 58 | ICMPv6 | ICMP for IPv6 (includes NDP) |
| 88 | EIGRP | Enhanced Interior Gateway Routing Protocol |
| 89 | OSPF | Open Shortest Path First |
| 103 | PIM | Protocol Independent Multicast |
| 112 | VRRP | Virtual Router Redundancy Protocol |
| 115 | L2TP | Layer 2 Tunneling Protocol (data) |
| 132 | SCTP | Stream Control Transmission Protocol |

---

## Well-Known TCP Ports

| Port | Protocol | Description |
|------|----------|-------------|
| 20 | FTP Data | File Transfer Protocol — data channel |
| 21 | FTP Control | File Transfer Protocol — control channel |
| 22 | SSH | Secure Shell; also SCP, SFTP |
| 23 | Telnet | Unencrypted remote login (avoid) |
| 25 | SMTP | Simple Mail Transfer Protocol |
| 43 | WHOIS | WHOIS directory service |
| 49 | TACACS+ | Terminal Access Controller Access-Control System Plus |
| 53 | DNS | Domain Name System (also UDP 53) |
| 80 | HTTP | Hypertext Transfer Protocol |
| 88 | Kerberos | Authentication |
| 110 | POP3 | Post Office Protocol v3 |
| 119 | NNTP | Network News Transfer Protocol |
| 135 | MS-RPC | Microsoft Remote Procedure Call |
| 139 | NetBIOS Session | SMB over NetBIOS |
| 143 | IMAP | Internet Message Access Protocol |
| 179 | BGP | Border Gateway Protocol |
| 389 | LDAP | Lightweight Directory Access Protocol |
| 443 | HTTPS | HTTP over TLS |
| 445 | SMB/CIFS | Server Message Block (Direct) |
| 465 | SMTPS | SMTP over TLS (implicit) |
| 514 | RSH | Remote Shell (TCP; NOT syslog which is UDP 514) |
| 587 | SMTP Submission | Email submission (STARTTLS) |
| 636 | LDAPS | LDAP over TLS |
| 830 | NETCONF | NETCONF over SSH |
| 853 | DNS over TLS | DoT |
| 993 | IMAPS | IMAP over TLS |
| 995 | POP3S | POP3 over TLS |
| 1433 | MS-SQL | Microsoft SQL Server |
| 1521 | Oracle DB | Oracle Database listener |
| 1723 | PPTP | Point-to-Point Tunneling Protocol |
| 2049 | NFS | Network File System |
| 3306 | MySQL | MySQL / MariaDB database |
| 3389 | RDP | Remote Desktop Protocol |
| 5432 | PostgreSQL | PostgreSQL database |
| 5900+ | VNC | Virtual Network Computing |
| 6443 | Kubernetes API | K8s API server (HTTPS) |
| 8080 | HTTP Alt | Common HTTP alternative |
| 8443 | HTTPS Alt | Common HTTPS alternative |
| 8088 | Splunk HEC | HTTP Event Collector |
| 8089 | Splunk Mgmt | Splunk management port |
| 8443 | HTTPS Alt | Various (vCenter, etc.) |
| 9997 | Splunk Fwd | Splunk forwarder receiving |

---

## Well-Known UDP Ports

| Port | Protocol | Description |
|------|----------|-------------|
| 53 | DNS | Domain Name System (also TCP 53) |
| 67 | DHCP Server | DHCP/BOOTP server |
| 68 | DHCP Client | DHCP/BOOTP client |
| 69 | TFTP | Trivial File Transfer Protocol |
| 123 | NTP | Network Time Protocol |
| 161 | SNMP | SNMP queries (GET/SET) |
| 162 | SNMP Trap | SNMP trap notifications |
| 443 | QUIC | HTTP/3 over QUIC |
| 500 | IKE | Internet Key Exchange (IPsec) |
| 514 | Syslog | Syslog messages (UDP; TCP 514 = RSH) |
| 546 | DHCPv6 Client | DHCPv6 client |
| 547 | DHCPv6 Server | DHCPv6 server/relay |
| 623 | IPMI | Intelligent Platform Management Interface |
| 646 | LDP | Label Distribution Protocol (also TCP 646) |
| 1194 | OpenVPN | OpenVPN (also TCP) |
| 1645/1646 | RADIUS (legacy) | Old RADIUS auth/acct ports |
| 1812 | RADIUS Auth | RADIUS authentication (standard) |
| 1813 | RADIUS Acct | RADIUS accounting (standard) |
| 1985 | HSRPv1 | Hot Standby Router Protocol v1 |
| 2029 | HSRPv2 | Hot Standby Router Protocol v2 |
| 2055 | NetFlow | Cisco NetFlow (common collector port) |
| 3222 | GLBP | Gateway Load Balancing Protocol |
| 3784 | BFD | Bidirectional Forwarding Detection (single-hop) |
| 3785 | BFD Multi-hop | BFD multihop |
| 4500 | IKE NAT-T | IKE with NAT Traversal (IPsec) |
| 4739 | IPFIX | IP Flow Information Export |
| 4789 | VXLAN | VXLAN data plane |
| 4790 | GENEVE | Generic Network Virtualization Encapsulation |
| 5246 | CAPWAP Control | CAPWAP control channel (DTLS) |
| 5247 | CAPWAP Data | CAPWAP data channel |
| 6081 | Geneve | Geneve encapsulation (alt) |
| 6343 | sFlow | sFlow collector |
| 8472 | OTV / VXLAN (Linux) | Linux kernel VXLAN (non-standard; prefer 4789) |
| 9995/9996 | NetFlow | Alternative NetFlow collector ports |
| 51820 | WireGuard | WireGuard VPN |

---

## Layer 2 Protocols (No TCP/UDP Port)

| Protocol | Identifier | Multicast Address | Purpose |
|----------|-----------|-------------------|---------|
| STP/RSTP/MST | BPDU | 01:80:C2:00:00:00 | Spanning Tree Protocol |
| LLDP | Ethertype 0x88CC | 01:80:C2:00:00:0E | Link Layer Discovery Protocol |
| CDP | SNAP/LLC | 01:00:0C:CC:CC:CC | Cisco Discovery Protocol |
| LACP | Ethertype 0x8809 | 01:80:C2:00:00:02 | Link Aggregation Control Protocol |
| 802.1X/EAP | Ethertype 0x888E | 01:80:C2:00:00:03 | Port-based authentication |
| MACsec | Ethertype 0x88E5 | — | Layer 2 encryption |
| ARP | Ethertype 0x0806 | — (broadcast) | Address Resolution Protocol |
| 802.1Q | Ethertype 0x8100 | — | VLAN tagging |
| QinQ | Ethertype 0x88A8 | — | Double VLAN tagging (802.1ad) |
| MPLS Unicast | Ethertype 0x8847 | — | MPLS label switching |
| MPLS Multicast | Ethertype 0x8848 | — | MPLS multicast |
| PPPoE Discovery | Ethertype 0x8863 | — | PPPoE discovery stage |
| PPPoE Session | Ethertype 0x8864 | — | PPPoE session stage |
| IPv4 | Ethertype 0x0800 | — | IP version 4 |
| IPv6 | Ethertype 0x86DD | — | IP version 6 |

---

## Multicast Address Assignments

### IPv4 Multicast (224.0.0.0/4)

| Address | Protocol |
|---------|----------|
| 224.0.0.1 | All hosts |
| 224.0.0.2 | All routers (also LDP discovery, HSRPv1) |
| 224.0.0.5 | All OSPF routers |
| 224.0.0.6 | All OSPF DRs |
| 224.0.0.9 | RIPv2 |
| 224.0.0.10 | EIGRP |
| 224.0.0.13 | PIM |
| 224.0.0.18 | VRRP |
| 224.0.0.22 | IGMPv3 |
| 224.0.0.102 | HSRPv2, GLBP |
| 224.0.0.251 | mDNS |
| 224.0.0.252 | LLMNR |
| 224.0.1.39 | Cisco RP-Announce |
| 224.0.1.40 | Cisco RP-Discovery |
| 239.0.0.0/8 | Administratively scoped (private multicast) |

### IPv4 Multicast Ranges
| Range | Scope |
|-------|-------|
| 224.0.0.0/24 | Link-local (TTL=1; not forwarded) |
| 224.0.1.0 - 238.255.255.255 | Global scope |
| 239.0.0.0/8 | Administratively scoped (organization-local) |

---

## ICMP Types and Codes

### ICMPv4

| Type | Code | Description |
|------|------|-------------|
| 0 | 0 | Echo Reply (ping response) |
| 3 | 0 | Destination Unreachable — Network unreachable |
| 3 | 1 | Destination Unreachable — Host unreachable |
| 3 | 3 | Destination Unreachable — Port unreachable |
| 3 | 4 | Destination Unreachable — Fragmentation needed + DF set (PMTUD) |
| 3 | 13 | Destination Unreachable — Administratively prohibited |
| 5 | 0 | Redirect — Network redirect |
| 8 | 0 | Echo Request (ping) |
| 11 | 0 | Time Exceeded — TTL expired (traceroute) |
| 11 | 1 | Time Exceeded — Fragment reassembly |

### ICMPv6

| Type | Description |
|------|-------------|
| 1 | Destination Unreachable |
| 2 | Packet Too Big (PMTUD) |
| 3 | Time Exceeded |
| 128 | Echo Request |
| 129 | Echo Reply |
| 133 | Router Solicitation (NDP) |
| 134 | Router Advertisement (NDP) |
| 135 | Neighbor Solicitation (NDP) |
| 136 | Neighbor Advertisement (NDP) |
| 137 | Redirect (NDP) |

---

## TCP/UDP Fundamentals

### TCP Header Key Fields
- Source Port (16-bit), Destination Port (16-bit)
- Sequence Number (32-bit), Acknowledgment Number (32-bit)
- Flags: SYN, ACK, FIN, RST, PSH, URG, ECE, CWR
- Window Size (16-bit; scaled with Window Scale option)
- Checksum (16-bit; mandatory)

### TCP Three-Way Handshake
1. Client → Server: SYN (seq=x)
2. Server → Client: SYN-ACK (seq=y, ack=x+1)
3. Client → Server: ACK (seq=x+1, ack=y+1)

### TCP Four-Way Teardown
1. A → B: FIN
2. B → A: ACK
3. B → A: FIN
4. A → B: ACK

### UDP Header
- Source Port (16-bit), Destination Port (16-bit)
- Length (16-bit), Checksum (16-bit; optional in IPv4, mandatory in IPv6)
- No connection state, no reliability, no ordering

### Ephemeral Port Ranges
| OS | Range |
|-----|-------|
| IANA recommended | 49152-65535 |
| Linux | 32768-60999 |
| Windows | 49152-65535 |
| BSD/macOS | 49152-65535 |
| Cisco IOS | 1024-65535 |
