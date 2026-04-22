# Network Management Platforms Deep Reference

## Cisco Catalyst Center (formerly DNA Center)

### Overview
On-premises network controller and management platform for Cisco enterprise networks. Provides intent-based networking, assurance, automation, and security.

### Core Functions
| Function | Description |
|----------|-------------|
| **Design** | Sites, floors, buildings hierarchy; IP pools; network profiles |
| **Policy** | Group-based access control (SGT/VN); application policies |
| **Provision** | PnP onboarding; templates; SWIM (Software Image Management) |
| **Assurance** | AI/ML analytics; client health; device health; application health |
| **Platform** | APIs; event notifications; ITSM integration; SDK |

### API Architecture
- **Intent API (Northbound)**: RESTful; 1000+ operations; HTTPS with token auth
- **Integration API (Westbound)**: ITSM integration (ServiceNow, etc.)
- **Events API (Eastbound)**: Webhooks for external event consumption
- **Multivendor SDK**: Extend support to non-Cisco devices

### API Authentication
```
POST https://<catalyst-center>/dna/system/api/v1/auth/token
Authorization: Basic <base64(user:pass)>

Response: {"Token": "eyJ..."}

Subsequent requests:
X-Auth-Token: eyJ...
```

### Key API Endpoints
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/dna/intent/api/v1/network-device` | GET | Device inventory |
| `/dna/intent/api/v1/client-health` | GET | Client health scores |
| `/dna/intent/api/v1/device-health` | GET | Device health scores |
| `/dna/intent/api/v1/issues` | GET | Assurance issues |
| `/dna/intent/api/v1/site` | GET/POST | Site hierarchy |
| `/dna/intent/api/v1/template-programmer/template` | GET/POST | Config templates |
| `/dna/intent/api/v1/image/importation` | POST | Software images |
| `/dna/intent/api/v2/pnp/device` | GET | PnP device status |

### Assurance Features
- **AI Network Analytics**: Baseline comparison, anomaly detection, peer comparison
- **Client 360**: Per-client journey (DHCP, AAA, DNS, association, roaming)
- **Device 360**: Per-device health (CPU, memory, interfaces, neighbors)
- **Application Health**: NBAR-based application recognition; SLA monitoring
- **Path Trace**: Visual path analysis between endpoints
- **Intelligent Capture**: On-demand packet capture from APs

### Supported Platforms
Catalyst 9000 switches, Catalyst 9800 WLCs, ISR 4000/1000 routers, Catalyst 8000 routers, Aironet/Catalyst APs

---

## Cisco Meraki Dashboard

### Overview
Cloud-managed networking for switches, APs, security appliances, cameras, and sensors. Single pane of glass at dashboard.meraki.com.

### Product Lines
| Product | Type | Examples |
|---------|------|---------|
| MR | Wireless APs | MR28, MR36, MR46, MR57, MR78 |
| MS | Switches | MS120, MS130, MS210, MS250, MS350, MS390, MS410, MS450 |
| MX | Security Appliances | MX64, MX67, MX68, MX75, MX85, MX95, MX105, MX250 |
| MV | Smart Cameras | MV2, MV12, MV22, MV32, MV52, MV72 |
| MT | Sensors | MT10 (temp), MT11 (temp+humidity), MT12 (water leak), MT14 (air quality), MT15 (PM2.5), MT30 (button), MT40 (plug) |
| MG | Cellular Gateways | MG21, MG41, MG51 |
| SM | Systems Manager (MDM) | — |

### API (Dashboard API v1)
- **Base URL**: `https://api.meraki.com/api/v1/`
- **Auth**: `Authorization: Bearer <API-KEY>`
- **Rate limit**: 10 requests/sec per org (burst to 30)
- **Pagination**: Link header for next page
- **OpenAPI v3 spec available**: enables SDK generation
- **Webhooks**: Real-time alerts to any HTTP endpoint
- **Action Batches**: Group multiple API calls into single transaction

### Key API Endpoints
| Endpoint | Purpose |
|----------|---------|
| `/organizations` | List organizations |
| `/organizations/{id}/networks` | List networks |
| `/networks/{id}/devices` | List devices in network |
| `/devices/{serial}/switchPorts` | Switch port configuration |
| `/networks/{id}/wireless/ssids` | SSID configuration |
| `/organizations/{id}/devices/statuses` | Device online/offline status |
| `/organizations/{id}/sensor/readings/history` | MT sensor data |
| `/networks/{id}/appliance/vpn/siteToSiteVpn` | Auto VPN config |

### Key Features
- **Auto VPN**: Automated full-mesh or hub-spoke VPN between MX appliances
- **Adaptive Policy**: Group Policy based on identity (SGT equivalent)
- **Auto RF**: Automatic channel and power optimization for wireless
- **Air Marshal**: Built-in WIDS/WIPS
- **SD-WAN**: Application-aware routing; active-active uplinks; traffic shaping
- **Health alerts**: Configurable email/webhook alerts on network events

---

## Cisco ThousandEyes

### Overview
Digital experience monitoring (DEM) platform. Provides network path visibility, application performance monitoring, and internet intelligence across owned and unowned networks.

### Agent Types
| Agent | Deployment | Use Case | Test Types |
|-------|-----------|----------|------------|
| Cloud Agent | ThousandEyes-managed; 200+ cities | Outside-in monitoring; SaaS visibility | All (except BGP as source) |
| Enterprise Agent | Customer-deployed (VM/container/appliance) | Inside-out; internal app monitoring | All except BGP |
| Endpoint Agent | Installed on Windows/Mac | Employee experience; browser-level visibility | HTTP, Network |

### Test Types
| Category | Test | Measures |
|----------|------|---------|
| Network | Agent-to-Server | Loss, latency, jitter, path MTU |
| Network | Agent-to-Agent | Bidirectional network performance |
| Network | Path Trace | Hop-by-hop path visualization |
| Routing | BGP | AS path changes, prefix hijacks, leaks |
| DNS | DNS Server | Resolution time, mapping accuracy |
| DNS | DNS Trace | Full recursive resolution path |
| DNS | DNSSEC | Validation chain |
| Web | HTTP Server | Response time, status codes, headers |
| Web | Page Load | DOM timing, waterfall, component load |
| Web | Transaction | Multi-step Selenium scripts |
| Voice | SIP Server | Registration, call setup |
| Voice | RTP Stream | MOS, latency, jitter, loss |

### Path Visualization
- Shows every L3 hop between agent and target
- Identifies ISP, cloud provider, CDN per hop
- Highlights packet loss, latency spikes, and MPLS tunnels
- Groups by agent, network, location, interface, or destination

### Integration Methods
| Method | Protocol | Use Case |
|--------|----------|----------|
| OpenTelemetry Streaming | HTTPS/HEC | Real-time metrics to Splunk/OTEL collector |
| REST API v7 | HTTPS | Pull test results, alerts, agent status |
| Webhooks | HTTPS | Push alert notifications |
| ServiceNow Integration | HTTPS | Incident correlation |
| PagerDuty Integration | HTTPS | Alert escalation |

### Key Differentiators
- Visibility into third-party networks (ISP, SaaS, cloud) where you have no control
- Internet Insights: aggregate intelligence across all ThousandEyes customers
- Catalyst Center integration: correlate internal assurance with external path data
- WAN Insights: SD-WAN overlay performance correlation

---

## HPE Aruba Central

### Overview
Cloud-native management platform for Aruba APs, switches (AOS-CX), and gateways. Provides unified monitoring, configuration, and AI-driven operations.

### Key Features
| Feature | Description |
|---------|-------------|
| **AI Insights** | ML-based anomaly detection, recommendations |
| **AIOps** | Automated root-cause analysis for WiFi issues |
| **Firmware Management** | Scheduled upgrades with compliance tracking |
| **Guest Management** | Captive portal, sponsor-based access |
| **Presence Analytics** | Location-based analytics (with Aruba APs) |
| **SD-Branch** | Unified branch networking (WAN + LAN + WiFi + security) |
| **WLAN Management** | Multi-site SSID, RF, policy management |
| **Switch Management** | AOS-CX switch configuration at scale |

### API
- REST API at `https://central.arubanetworks.com/api/`
- OAuth 2.0 authentication (token + refresh)
- Webhooks for alerts
- Streaming API for real-time telemetry

### Deployment Models
- **Aruba Central (cloud)**: SaaS; fully hosted
- **Aruba Central On-Premises**: Self-hosted for air-gapped environments

---

## Juniper Mist

### Overview
Cloud-native, AI-driven network management. Uses Mist AI engine and Marvis Virtual Network Assistant for automated operations.

### Key Features
| Feature | Description |
|---------|-------------|
| **Marvis VNA** | AI assistant for natural-language troubleshooting |
| **Wired Assurance** | Extends AI-driven ops to Juniper EX switches |
| **WAN Assurance** | Extends to Juniper SRX/SSR WAN routers |
| **WiFi Assurance** | SLE (Service Level Expectation) framework |
| **vBLE** | Virtual Bluetooth LE for indoor location without overlay |
| **Auto-Provisioning** | Cloud-based ZTP for APs, switches, routers |
| **Marvis Actions** | Auto-remediation of detected issues |

### Service Level Expectations (SLEs)
Mist defines 7 wireless SLEs:
1. **Time to Connect** — Association + authentication time
2. **Throughput** — Actual client throughput
3. **Successful Connect** — Success rate of connection attempts
4. **Roaming** — Roaming success and time
5. **Coverage** — Signal quality (RSSI)
6. **Capacity** — AP utilization
7. **AP Availability** — AP uptime

### API
- REST API at `https://api.mist.com/api/v1/`
- API Token or OAuth 2.0
- WebSocket for real-time events
- Terraform provider available

### Integration
- ITSM: ServiceNow integration
- Monitoring: Webhooks, syslog forwarding
- SIEM: Syslog, webhook to Splunk/SIEM
- Location: vBLE + SDK for custom apps

---

## Arista CloudVision (CVP/CVaaS)

### Overview
Network-wide workload orchestration and telemetry platform for Arista EOS switches. Available as on-prem appliance (CVP) or cloud service (CVaaS).

### Key Features
| Feature | Description |
|---------|-------------|
| **Network Telemetry** | Streaming telemetry via TerminAttr agent on EOS |
| **Change Control** | Workflow-based config changes with approval gates |
| **Compliance** | Image and config compliance dashboards |
| **Topology** | Real-time network topology visualization |
| **Studios** | Declarative network provisioning (intent-based) |
| **Events** | Centralized event/alert management |
| **Configlets** | Reusable config snippets for standardization |
| **Image Management** | Firmware lifecycle management |

### TerminAttr Agent
- Runs on every Arista EOS switch
- Streams state data (interfaces, BGP, OSPF, MLAG, VXLAN) to CVP/CVaaS
- Uses gNMI (gRPC Network Management Interface) or OpenConfig
- Config: `daemon TerminAttr` with CVP/CVaaS address

### API
- RESTful API and Resource APIs
- gNMI for streaming telemetry
- eAPI on individual switches (JSON-RPC over HTTPS)

### Studios (Declarative Provisioning)
- Define network intent (VLANs, routing, VXLAN) as templates
- CloudVision generates device-specific configs
- Change control workflow for review and approval
- Compliance monitoring ensures drift detection

---

## Network Automation Protocols

### NETCONF (RFC 6241)
- **Transport**: SSH (TCP 830)
- **Data encoding**: XML
- **Data models**: YANG
- **Operations**: get, get-config, edit-config, copy-config, delete-config, lock, unlock, commit
- **Supported by**: Cisco IOS-XE/IOS-XR/NX-OS, Juniper, Arista, Aruba

### RESTCONF (RFC 8040)
- **Transport**: HTTPS
- **Data encoding**: XML or JSON
- **Data models**: YANG (same as NETCONF)
- **Operations**: HTTP verbs (GET, POST, PUT, PATCH, DELETE)
- **Simpler than NETCONF**: no SSH session; stateless

### gNMI (gRPC Network Management Interface)
- **Transport**: gRPC over HTTP/2 (TLS)
- **Data encoding**: Protobuf or JSON
- **Data models**: OpenConfig YANG
- **Operations**: Get, Set, Subscribe (streaming telemetry)
- **Primary use**: High-performance streaming telemetry

### YANG (RFC 7950)
- Data modeling language for NETCONF/RESTCONF/gNMI
- **Types**: Native vendor models (Cisco-IOS-XE-*, Juniper-*, Arista-*) and OpenConfig models
- OpenConfig models: vendor-neutral; preferred for multi-vendor automation
- YANG Explorer / pyang tools for model browsing

### Model-Driven Telemetry
| Protocol | Encoding | Frequency | Direction | Use Case |
|----------|----------|-----------|-----------|----------|
| gNMI Subscribe | Protobuf | On-change or periodic | Device → Collector | Real-time streaming |
| NETCONF notification | XML | Event-driven | Device → Collector | State change events |
| Dial-in (gNMI) | Protobuf | On request | Collector → Device | Pull-based |
| Dial-out (gRPC) | Protobuf/JSON | Periodic | Device → Collector | Push-based (Cisco) |
