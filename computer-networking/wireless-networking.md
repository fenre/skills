# Wireless Networking Deep Reference

## WiFi Standards

| Standard | WiFi Gen | Year | Bands | Max Rate | Max Width | Modulation | MIMO | Key Feature |
|----------|---------|------|-------|----------|-----------|------------|------|-------------|
| 802.11a | — | 1999 | 5 GHz | 54 Mbps | 20 MHz | 64-QAM | — | First 5 GHz |
| 802.11b | — | 1999 | 2.4 GHz | 11 Mbps | 22 MHz | CCK | — | — |
| 802.11g | — | 2003 | 2.4 GHz | 54 Mbps | 20 MHz | 64-QAM | — | — |
| 802.11n | WiFi 4 | 2009 | 2.4/5 GHz | 600 Mbps | 40 MHz | 64-QAM | 4x4 SU | Channel bonding, MIMO |
| 802.11ac | WiFi 5 | 2013 | 5 GHz | 6.9 Gbps | 160 MHz | 256-QAM | 8x8 DL MU | Beamforming, MU-MIMO DL |
| 802.11ax | WiFi 6 | 2020 | 2.4/5 GHz | 9.6 Gbps | 160 MHz | 1024-QAM | 8x8 UL+DL MU | OFDMA, BSS Color, TWT |
| 802.11ax | WiFi 6E | 2021 | 2.4/5/6 GHz | 9.6 Gbps | 160 MHz | 1024-QAM | 8x8 UL+DL MU | 6 GHz band (1200 MHz) |
| 802.11be | WiFi 7 | 2024 | 2.4/5/6 GHz | 46 Gbps | 320 MHz | 4096-QAM | 16x16 MU | MLO, 320 MHz, puncturing |

### Key WiFi 6 Features
- **OFDMA**: Divides channel into Resource Units (RUs); serves multiple clients simultaneously
- **BSS Color**: 6-bit identifier to reduce co-channel interference (CCI) between overlapping BSSs
- **TWT (Target Wake Time)**: Scheduled wake times for IoT/battery devices
- **MU-MIMO**: Both uplink and downlink (WiFi 5 was DL only)
- **1024-QAM**: 25% throughput increase over 256-QAM

### Key WiFi 7 Features
- **MLO (Multi-Link Operation)**: Device connects on 2+ bands simultaneously; aggregates or fails over
- **4096-QAM**: 20% throughput increase over 1024-QAM
- **320 MHz channels**: Available in 6 GHz only; doubles WiFi 6E max width
- **Preamble Puncturing**: Use 320 MHz channel even if part is occupied (puncture the busy sub-channel)
- **Multi-RU**: A single client can be assigned multiple non-contiguous RUs

---

## Channel Planning

### 2.4 GHz Band
- 14 channels total; channels 1-11 usable in US/Canada; 1-13 in most of world
- Channel width: 20 MHz (22 MHz actual)
- **Non-overlapping channels (20 MHz): 1, 6, 11** (the only valid combination for US)
- Alternate: 1, 5, 9, 13 (in regions allowing ch 13; less common)
- 40 MHz bonding possible but NOT recommended in enterprise (only 1 non-overlapping pair)

### 5 GHz Band (UNII bands)
| Band | Channels (20 MHz) | Frequency | DFS | Notes |
|------|-------------------|-----------|-----|-------|
| UNII-1 | 36, 40, 44, 48 | 5.180-5.240 GHz | No | Indoor only (most regions) |
| UNII-2 | 52, 56, 60, 64 | 5.260-5.320 GHz | Yes | DFS/TPC required |
| UNII-2 Extended | 100-144 (varies) | 5.500-5.720 GHz | Yes | DFS/TPC; channel 144 varies |
| UNII-3 | 149, 153, 157, 161, 165 | 5.745-5.825 GHz | No | Highest power allowed |

- **Non-overlapping 20 MHz channels**: up to 25 (varies by regulatory domain)
- **40 MHz**: up to 12 non-overlapping
- **80 MHz**: up to 6 non-overlapping
- **160 MHz**: up to 2-3 non-overlapping (contiguous) or more with 80+80

### 6 GHz Band (WiFi 6E/7)
- **1200 MHz** of spectrum (5.925-7.125 GHz)
- **59 channels** at 20 MHz
- **29 channels** at 40 MHz
- **14 channels** at 80 MHz
- **7 channels** at 160 MHz
- **3 channels** at 320 MHz (WiFi 7 only)
- Preferred Starting Frequencies (PSC): channels 5, 21, 37, 53, 69, 85, 101, 117, 133, 149, 165, 181, 197, 213, 229 (used for discovery)
- Indoor-only (Low Power Indoor) standard power requires AFC (Automated Frequency Coordination)

---

## Wireless Controller Architectures

### Cisco Wireless
| Platform | Type | Scale | Management |
|----------|------|-------|------------|
| Cisco 9800-CL | Virtual (ESXi/KVM/Cloud) | Up to 6000 APs | On-prem / Cloud |
| Cisco 9800-40 | Appliance | 6000 APs / 64K clients | On-prem |
| Cisco 9800-80 | Appliance | 6000 APs / 64K clients | On-prem |
| Cisco 9800-L | Appliance (small) | 250-500 APs | On-prem |
| Catalyst Center embedded | Embedded WLC | 200 APs | Catalyst Center |
| Meraki Cloud | Cloud-managed | Unlimited (per license) | Meraki Dashboard |

**Cisco AP modes:**
- **Local**: Tunnel all traffic to WLC (CAPWAP); standard deployment
- **FlexConnect**: Local switching at branch; survives WLC failure
- **Monitor**: Dedicated RF monitoring (wIPS, rogue detection)
- **Sniffer**: Captures packets and sends to analyzer
- **Bridge**: Point-to-point/multipoint wireless bridge (mesh)
- **Flex+Bridge**: FlexConnect + Mesh
- **Sensor**: Dedicated synthetic testing (not serving clients)

**CAPWAP**: Control And Provisioning of Wireless Access Points (RFC 5415)
- Control channel: UDP 5246 (DTLS encrypted)
- Data channel: UDP 5247 (optionally DTLS encrypted)

### Cisco Meraki Wireless
- Fully cloud-managed via dashboard.meraki.com
- APs connect to Meraki cloud for management/config; data forwarded locally (NAT mode) or tunneled
- Auto RF: Automatic channel and power optimization
- Air Marshal: Built-in WIDS/WIPS (rogue detection, containment)
- Models: MR series (MR28, MR36, MR46, MR57 for WiFi 6E, MR78 for WiFi 7)
- API: REST API v1 at api.meraki.com

### HPE Aruba Wireless
| Platform | Type | Notes |
|----------|------|-------|
| Aruba Central | Cloud management | Multi-vendor; APs + switches + gateways |
| Aruba Mobility Controller (MC) | On-prem controller | 7200 series; hardware controllers |
| Aruba Gateway | SD-Branch | Combines WLC + SD-WAN + firewall |
| Aruba InstantAP | Controller-less | Virtual controller elected among APs |

**Aruba AP modes:**
- **Campus AP (CAP)**: Controller-managed (tunnel mode)
- **Remote AP (RAP)**: Branch; survives controller failure
- **Instant AP (IAP)**: Controller-less cluster; virtual controller
- **Mesh**: Wireless backhaul

**Key Aruba differentiators:**
- **AirMatch**: ML-based RF optimization (channel, power, bandwidth)
- **ClientMatch**: Steers clients to best AP based on capability
- **Dynamic Segmentation**: Assigns user role/VLAN based on identity
- **UXI (User Experience Insight)**: Synthetic testing sensors

### Juniper Mist Wireless
- **Cloud-native**: Mist AI engine; managed via mist.com
- **AI-driven**: Marvis Virtual Network Assistant for root-cause analysis
- **vBLE (Virtual Bluetooth LE)**: Built-in location services without separate overlay
- **APs**: AP12, AP32, AP33, AP34, AP43, AP45, AP63, AP64 (WiFi 6/6E/7)
- **Wired Assurance**: Extends to Juniper EX switches
- **WAN Assurance**: Extends to Juniper SRX/SSR

---

## Wireless Design Best Practices

### RF Design
- **Cell overlap**: 15-20% for seamless roaming (higher for voice/location)
- **Channel reuse**: Minimize co-channel interference; separate same-channel APs by 2+ cells
- **Power levels**: Start low; let controller optimize. Avoid max power (creates sticky clients)
- **Band steering**: Push 5 GHz/6 GHz capable clients off 2.4 GHz
- **Minimum RSSI**: -67 dBm for voice, -72 dBm for data
- **SNR**: Minimum 25 dB for voice, 20 dB for data

### Roaming Protocols
| Protocol | Standard | Key Feature | Latency |
|----------|----------|-------------|---------|
| 802.11r (FT) | IEEE | Fast BSS Transition; pre-authenticates | <50ms |
| 802.11k | IEEE | Neighbor report; AP publishes neighbor list | Aids roam decision |
| 802.11v | IEEE | BSS Transition Management; AP suggests target | Aids roam decision |
| OKC | Cisco/industry | Opportunistic Key Caching; PMK cached across APs | <50ms |
| CCKM | Cisco | Cisco Centralized Key Management | <50ms |

### Antenna Types
- **Omnidirectional**: 360-degree horizontal coverage; typical for indoor ceiling mount
- **Directional (patch/panel)**: Focused beam; used for hallways, outdoor point-to-multipoint
- **Sector**: Wide beam in one direction; used for outdoor/stadium
- **Yagi**: Highly directional; point-to-point bridges

### Deployment Models
| Model | Use Case | APs Per Floor | Density |
|-------|----------|---------------|---------|
| Coverage-based | Office, general | 1 per 2500-5000 sq ft | Low |
| Capacity-based | Conference rooms, auditoriums | 1 per 20-30 users | High |
| Location-based | Asset tracking, wayfinding | 1 per 2500 sq ft + overlap | Medium-High |
| Outdoor | Campus, warehouse | Varies; directional antennas | Low-Medium |
