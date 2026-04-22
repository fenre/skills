---
name: cisco-splunk-integration
description: "Comprehensive guide for integrating Cisco products with Splunk for security monitoring, network visibility, and operational intelligence. Covers: (1) Cisco Meraki - cloud networking, sensors, cameras, (2) Cisco ISE - identity and access management, (3) Cisco SD-WAN - software-defined WAN security, (4) Cisco Catalyst Center - network assurance, (5) Cisco ThousandEyes - digital experience monitoring, (6) Cisco Cyber Vision - OT/ICS security, (7) Cisco Secure Firewall - FTD, ASA, eStreamer, (8) Cisco Webex - collaboration and meetings, (9) Cisco Spaces - indoor location services, (10) Cisco Edge Intelligence - IoT data orchestration. Use when configuring data exports, understanding use cases, or building dashboards for Cisco products in Splunk."
---

# Cisco Products Integration with Splunk

## Overview

This skill covers the integration of major Cisco products with Splunk, including use cases, data types, and export methods for each product family.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CISCO PRODUCT FAMILIES                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  NETWORKING          │  SECURITY            │  OBSERVABILITY                │
│  ─────────────────   │  ─────────────────   │  ─────────────────            │
│  • Meraki            │  • ISE               │  • ThousandEyes               │
│  • Catalyst Center   │  • Secure Firewall   │  • Cyber Vision               │
│  • SD-WAN            │  • Secure Endpoint   │                               │
│  • Spaces            │                      │                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  COLLABORATION       │  IOT / INDUSTRIAL                                    │
│  ─────────────────   │  ─────────────────                                   │
│  • Webex Meetings    │  • Edge Intelligence                                 │
│  • Webex Calling     │  • Meraki Sensors                                    │
│  • Webex Contact Ctr │                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         EXPORT METHODS                                       │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐ │
│  │   REST    │  │  Syslog   │  │  Webhook  │  │ eStreamer │  │   MQTT    │ │
│  │   API     │  │  TCP/UDP  │  │   HEC     │  │  (FTD)    │  │  (IoT)    │ │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘  └───────────┘ │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐                               │
│  │ Firehose  │  │  OPC-UA   │  │  Modbus   │                               │
│  │ (Spaces)  │  │  (EI)     │  │  (EI)     │                               │
│  └───────────┘  └───────────┘  └───────────┘                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SPLUNK PLATFORM                                      │
│  Add-ons (TA)  │  Apps  │  Data Models  │  CIM Mapping  │  Dashboards      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Cisco Meraki

### Product Overview
Cloud-managed networking platform including wireless APs, switches, security appliances, cameras, and environmental sensors.

### Use Cases

| Use Case | Description | Data Sources |
|----------|-------------|--------------|
| **Network Monitoring** | Device health, uptime, connectivity | Device status, uplink health |
| **Wireless Analytics** | Client usage, RF health, roaming | Client statistics, wireless health |
| **Security Events** | Threat detection, Air Marshal alerts | Security events, IDS alerts |
| **Environmental Monitoring** | Temperature, humidity, air quality | MT sensor readings |
| **Smart Camera Analytics** | People counting, motion detection | MV Sense API, MQTT streams |
| **Energy Management** | Switch power consumption, PoE usage | Switch power history |
| **License Management** | License compliance, expiration tracking | License overview, entitlements |

### Data Types Created

| Sourcetype | Description | Key Fields |
|------------|-------------|------------|
| `meraki:devices` | Device inventory and status | serial, model, networkId, status |
| `meraki:sensorreadingshistory` | MT sensor telemetry | temperature, humidity, tvoc, pm25 |
| `meraki:webhook` | Real-time network alerts | alertType, alertLevel, deviceSerial |
| `meraki_mt_json` | MT sensor MQTT data | serial, metric, value, ts |
| `meraki_mv_json` | MV camera analytics | entrances, ts, zoneId |
| `meraki:apirequestshistory` | API usage tracking | responseCode, userAgent, ts |
| `meraki:firmwareupgrades` | Firmware status | currentVersion, status |

### Export Methods

**1. REST API (Primary)**
```bash
# API endpoint structure
https://api.meraki.com/api/v1/organizations/{orgId}/...

# Authentication header
Authorization: Bearer YOUR_API_KEY
```

**Splunk Add-on:** Cisco Meraki Add-on for Splunk
- Polls REST API endpoints on configurable intervals
- Supports organization, network, and device-level data collection
- CIM-compatible field mappings

**2. Webhooks (Real-time Alerts)**
```
Meraki Dashboard → Organizations → API & Webhooks → Webhooks
→ Configure HEC endpoint URL
→ Select alert types to forward
```

Webhook payload example:
```json
{
  "version": "0.1",
  "sharedSecret": "secret",
  "sentAt": "2024-01-15T10:30:00Z",
  "organizationId": "123456",
  "alertType": "Settings changed",
  "alertLevel": "informational",
  "alertData": {...}
}
```

**3. MQTT (IoT Sensors & Cameras)**
```
# MT Sensors → MQTT Broker → Splunk Edge Hub → Splunk
# MV Cameras → MQTT (MV Sense API) → Splunk Edge Hub → Splunk

Topic patterns:
- meraki/v1/mt/{serial}/ble/{mac}/raw  # MT sensor BLE
- merakimv/{serial}/raw_detections     # MV camera detections
- merakimv/{serial}/zones/{zoneId}     # Zone analytics
```

**4. Syslog (MX Security Appliance)**
```
Dashboard → Network-wide → General → Reporting
→ Add syslog server (Splunk IP:514)
→ Select log categories
```

### Splunk Apps & Add-ons

| Component | Splunkbase ID | Purpose |
|-----------|---------------|---------|
| Cisco Meraki Add-on for Splunk | 5580 | Data collection via API/webhooks |
| Cisco Enterprise Networking for Splunk | 7539 | Dashboards and visualizations |

### Sample SPL Queries

```spl
# Device availability overview
index=meraki sourcetype="meraki:devices"
| stats latest(status) as status by serial, name, model
| eval status_color = case(status="online", "green", status="offline", "red", true(), "yellow")

# Sensor readings analysis
index=meraki sourcetype="meraki:sensorreadingshistory"
| timechart span=1h avg(temperature) as avg_temp, avg(humidity) as avg_humidity by serial

# Security alerts
index=meraki sourcetype="meraki:webhook" alertLevel="critical"
| stats count by alertType, deviceSerial
| sort - count

# Camera people counting
index=meraki sourcetype=meraki_mv_json
| timechart span=1h sum(entrances) as total_entrances by serial
```

---

## Cisco Identity Services Engine (ISE)

### Product Overview
Security policy management platform for network access control, identity-based segmentation, and guest access.

### Use Cases

| Use Case | Description | Data Sources |
|----------|-------------|--------------|
| **Authentication Monitoring** | Track successful/failed authentications | RADIUS logs |
| **Posture Assessment** | Endpoint compliance verification | Posture events |
| **Guest Access Tracking** | Guest portal usage, sponsor activity | Guest logs |
| **Profiling Analytics** | Device type identification | Profiler events |
| **Security Group Tagging** | SGT assignment and changes | TrustSec events |
| **Admin Audit** | Configuration changes, admin activity | Audit logs |
| **Threat Response** | Adaptive Network Control (ANC) | pxGrid, EPS events |

### Data Types Created

| Sourcetype | Description | Key Fields |
|------------|-------------|------------|
| `cisco:ise:syslog` | All ISE syslog events | MESSAGE_CODE, user, nas_ip, action |
| `cisco:ise:audit` | Administrative changes | admin, object, action, result |
| `cisco:ise:radius` | RADIUS authentication | user, calling_station_id, nas_port |
| `cisco:ise:eps` | Endpoint protection events | mac, action, policy |

### Export Methods

**1. Syslog (Primary)**
```
ISE → Administration → System → Logging → Remote Logging Targets
→ Add new target (Splunk IP, TCP/UDP 514)
→ Map logging categories to target
```

Logging categories to enable:
- Passed Authentications
- Failed Attempts
- RADIUS Diagnostics
- Posture and Client Provisioning Audit
- Profiler

**2. pxGrid (Real-time Context)**
```
ISE → Administration → pxGrid Services
→ Enable pxGrid
→ Approve Splunk as pxGrid client
```

pxGrid capabilities:
- Real-time session data
- Endpoint profiles
- Security Group Tags (SGTs)
- Adaptive Network Control (quarantine/unquarantine)

**3. Data Connect (ISE 3.2+)**
```
ISE → Administration → System → Settings → Data Connect
→ Enable Data Connect
→ Export certificate for Splunk DB Connect
```

Direct database queries via JDBC:
```sql
SELECT access_service, sum(passed_count) as passed, 
       sum(failed_count) as failed
FROM radius_authentication_summary
GROUP BY access_service
```

### Splunk Apps & Add-ons

| Component | Splunkbase ID | Purpose |
|-----------|---------------|---------|
| Splunk Add-on for Cisco ISE | 1915 | Syslog data collection |
| Splunk for Cisco ISE | 1589 | Dashboards and reports |
| Cisco Catalyst Add-on | 7538 | Multi-product collection |

### Sample SPL Queries

```spl
# Authentication success/failure ratio
index=ise sourcetype="cisco:ise:syslog" 
| eval result=if(match(MESSAGE_CODE, "^5200"), "passed", "failed")
| stats count by result, user
| xyseries user result count

# Failed authentication by NAS
index=ise sourcetype="cisco:ise:syslog" MESSAGE_CODE=5400 OR MESSAGE_CODE=5401
| stats count as failures by nas_ip_address
| sort - failures | head 10

# Endpoint profiling distribution
index=ise sourcetype="cisco:ise:syslog" MESSAGE_CODE=80002
| stats dc(calling_station_id) as devices by EndPointMatchedProfile
| sort - devices

# Admin configuration changes
index=ise sourcetype="cisco:ise:audit"
| stats count by admin, object_type, action
| sort - count
```

---

## Cisco SD-WAN (Catalyst SD-WAN)

### Product Overview
Software-defined WAN solution for secure, optimized connectivity across branch offices, data centers, and cloud.

### Use Cases

| Use Case | Description | Data Sources |
|----------|-------------|--------------|
| **Security Event Monitoring** | IPS, malware, URL filtering events | Security syslogs |
| **VPN Performance** | Tunnel health, latency, jitter | VPN statistics |
| **Application Visibility** | App usage, DPI classification | NetFlow, AppQoE |
| **Policy Compliance** | Access control policy hits | Firewall logs |
| **Device Health** | CPU, memory, interface status | Device telemetry |
| **Traffic Analytics** | Bandwidth utilization, top talkers | NetFlow v9 |

### Data Types Created

| Sourcetype | Description | Key Fields |
|------------|-------------|------------|
| `cisco:sdwan:syslog` | General SD-WAN syslogs | host, severity, message |
| `cisco:sdwan:ips` | Intrusion prevention events | signature, action, src_ip |
| `cisco:sdwan:urlf` | URL filtering events | url, category, action |
| `cisco:sdwan:malware` | Malware detection events | file_hash, verdict, action |
| `cisco:sdwan:firewall` | Firewall policy events | rule, action, protocol |
| `stream:cisco_hsl_netflow` | NetFlow records | src_ip, dest_ip, bytes |

### Export Methods

**1. Syslog (Security Events)**
```
vManage → Administration → Settings → Logging
→ Add Splunk as syslog server
→ Configure severity levels and categories
```

Categories available:
- Security (IPS, malware, UTD)
- System (device health, alarms)
- Audit (configuration changes)

**2. NetFlow v9 (Traffic Analytics)**
```
vManage → Configuration → Templates → Feature Template
→ Enable NetFlow export
→ Configure collector (Splunk) IP and port 4739
```

Requires:
- Splunk App for Stream
- Splunk Add-on for Stream Forwarders
- Cisco SD-WAN HSL Add-on for Splunk

**3. vManage REST API**
```bash
# Authentication
POST https://vmanage-ip/j_security_check
Content-Type: application/x-www-form-urlencoded
j_username=admin&j_password=password

# Device statistics
GET https://vmanage-ip/dataservice/device/statistics
```

### Splunk Apps & Add-ons

| Component | Splunkbase ID | Purpose |
|-----------|---------------|---------|
| Cisco Catalyst SD-WAN Add-on | 6656 | Syslog/NetFlow collection |
| Cisco Catalyst SD-WAN App | 6657 | Security dashboards |
| Cisco Catalyst Add-on | 7538 | Multi-product collection |

### Sample SPL Queries

```spl
# Security events overview
index=sdwan sourcetype="cisco:sdwan*"
| stats count by sourcetype, action
| sort - count

# Top blocked IPS signatures
index=sdwan sourcetype="cisco:sdwan:ips" action="drop"
| stats count by signature_name, src_ip
| sort - count | head 20

# VPN tunnel health
index=sdwan sourcetype="cisco:sdwan:syslog" "tunnel"
| stats count by host, message
| where match(message, "down|flap|timeout")

# Application bandwidth usage (NetFlow)
index=sdwan sourcetype="stream:cisco_hsl_netflow"
| stats sum(bytes) as total_bytes by app_name
| eval GB = round(total_bytes/1024/1024/1024, 2)
| sort - GB
```

---

## Cisco Catalyst Center (formerly DNA Center)

### Product Overview
Network controller providing automation, assurance, and analytics for enterprise networks.

### Use Cases

| Use Case | Description | Data Sources |
|----------|-------------|--------------|
| **Network Assurance** | Client health, device health scores | Assurance API |
| **Issue Detection** | AI-driven anomaly alerts | Issues API |
| **Software Compliance** | Image version tracking | SWIM API |
| **Configuration Audit** | Configuration drift detection | Audit logs |
| **Wireless Performance** | RF health, interference detection | Wireless health |
| **Application Experience** | App health scores | AppQoE metrics |

### Data Types Created

| Sourcetype | Description | Key Fields |
|------------|-------------|------------|
| `cisco:dnac:issue` | Assurance issues/alerts | issueId, priority, category |
| `cisco:dnac:device` | Device inventory | hostname, platformId, health |
| `cisco:dnac:client` | Client health data | macAddress, healthScore |
| `cisco:dnac:audit` | Admin audit events | user, action, details |

### Export Methods

**1. REST API (Intent API)**
```bash
# Authentication - get token
POST https://catalyst-center/dna/system/api/v1/auth/token
Authorization: Basic base64(username:password)

# Device health
GET https://catalyst-center/dna/intent/api/v1/device-health

# Client health  
GET https://catalyst-center/dna/intent/api/v1/client-health

# Issues
GET https://catalyst-center/dna/intent/api/v1/issues
```

**2. Event Notifications (Webhooks)**
```
Catalyst Center → Platform → Developer Toolkit → Event Notifications
→ Create webhook subscription
→ Configure Splunk HEC endpoint
```

Event types:
- Assurance issues
- SWIM notifications
- PnP events
- Security advisories

**3. Syslog**
```
Catalyst Center → System → Settings → External Services → Destinations
→ Add syslog destination
```

### Splunk Apps & Add-ons

| Component | Splunkbase ID | Purpose |
|-----------|---------------|---------|
| Cisco Catalyst Center Add-on | 7858 | API data collection |
| Cisco DNA Center Add-on | 6668 | Legacy API collection |
| Cisco Catalyst Add-on | 7538 | Multi-product collection |

### Sample SPL Queries

```spl
# Device health overview
index=dnac sourcetype="cisco:dnac:device"
| stats latest(overallHealth) as health by hostname, platformId
| eval health_status = case(health>=80,"healthy", health>=50,"warning", true(),"critical")

# Critical issues
index=dnac sourcetype="cisco:dnac:issue" priority="P1"
| stats count by name, category
| sort - count

# Client connectivity issues
index=dnac sourcetype="cisco:dnac:client" healthScore<50
| stats count by ssid, band
| sort - count

# Assurance issue trends
index=dnac sourcetype="cisco:dnac:issue"
| timechart span=1h count by priority
```

---

## Cisco ThousandEyes

### Product Overview
Digital experience monitoring platform providing visibility into network paths, application performance, and user experience across owned and unowned networks.

### Use Cases

| Use Case | Description | Data Sources |
|----------|-------------|--------------|
| **Network Path Analysis** | Hop-by-hop path visibility | Path trace tests |
| **Application Performance** | Page load, transaction times | HTTP server tests |
| **DNS Monitoring** | Resolution times, failures | DNS tests |
| **VoIP Quality** | Latency, jitter, MOS scores | Voice tests |
| **Cloud Performance** | SaaS app reachability | Agent-to-server tests |
| **BGP Monitoring** | Route changes, hijacks | BGP tests |
| **Endpoint Experience** | End-user browser performance | Endpoint agents |

### Data Types Created

| Sourcetype | Description | Key Fields |
|------------|-------------|------------|
| `cisco:thousandeyes:test` | Test results | testName, metrics, agentId |
| `cisco:thousandeyes:alert` | Alert notifications | alertType, severity, rule |
| `cisco:thousandeyes:path` | Path visualization data | hops, latency, loss |
| `ThousandEyesOTel` | OpenTelemetry metrics | metric_name, value |

### Export Methods

**1. OpenTelemetry Streaming (Recommended)**
```
ThousandEyes → Integrations → Integrations 2.0
→ New Connector → Splunk Cloud Platform HEC / Splunk Enterprise HEC
→ Configure endpoint URL and HEC token
```

Stream configuration:
```json
{
  "type": "splunk-hec",
  "streamEndpointUrl": "https://http-inputs-{host}.splunkcloud.com:443/services/collector/event",
  "exporterConfig": {
    "token": "your-hec-token",
    "source": "ThousandEyesOTel",
    "sourceType": "ThousandEyesOTel",
    "index": "thousandeyes"
  }
}
```

**2. REST API**
```bash
# Authentication
Authorization: Bearer YOUR_OAUTH_TOKEN

# Test results
GET https://api.thousandeyes.com/v7/test-results/{testId}

# Alerts
GET https://api.thousandeyes.com/v7/alerts
```

**3. Webhooks (Alerts)**
```
ThousandEyes → Alerts → Alert Rules → Notifications
→ Add webhook → Configure Splunk HEC endpoint
```

### Splunk Apps & Add-ons

| Component | Splunkbase ID | Purpose |
|-----------|---------------|---------|
| Cisco ThousandEyes App for Splunk | 7719 | Dashboards and data collection |

### Sample SPL Queries

```spl
# Network latency by agent
index=thousandeyes sourcetype="ThousandEyesOTel"
| where metric_name="network.latency"
| timechart span=5m avg(value) by agent_name

# Test failures
index=thousandeyes sourcetype="cisco:thousandeyes:test"
| where status="failed"
| stats count by testName, agentName
| sort - count

# Path visualization
index=thousandeyes sourcetype="cisco:thousandeyes:path"
| mvexpand hops
| stats avg(latency) as avg_latency, avg(loss) as avg_loss by hop_ip
| sort avg_latency

# Alert correlation
index=thousandeyes sourcetype="cisco:thousandeyes:alert"
| timechart span=1h count by alertType
```

---

## Cisco Cyber Vision

### Product Overview
OT/ICS security solution providing visibility, threat detection, and segmentation for industrial networks.

### Use Cases

| Use Case | Description | Data Sources |
|----------|-------------|--------------|
| **Asset Discovery** | Industrial device inventory | Discovery events |
| **Vulnerability Management** | CVE tracking, risk scoring | Vulnerability data |
| **Threat Detection** | Anomaly detection, IDS alerts | Security events |
| **Network Segmentation** | IEC 62443 zones and conduits | Zone definitions |
| **Protocol Analysis** | Industrial protocol inspection | Flow data |
| **Compliance Reporting** | NERC CIP, IEC 62443 compliance | Audit logs |

### Data Types Created

| Sourcetype | Description | Key Fields |
|------------|-------------|------------|
| `cisco:cybervision:components` | Asset inventory | asset_name, vendor, model |
| `cisco:cybervision:flows` | Network flows | src_ip, dest_ip, protocol |
| `cisco:cybervision:events` | Security events | event_type, severity, asset |
| `cisco:cybervision:vulnerabilities` | CVE data | cve_id, cvss_score, asset |

### Export Methods

**1. REST API (Primary)**
```bash
# Authentication
POST https://cyber-vision-center/api/3.0/auth
Content-Type: application/json
{"username": "admin", "password": "password"}

# Components (Assets)
GET https://cyber-vision-center/api/3.0/components

# Events
GET https://cyber-vision-center/api/3.0/events

# Vulnerabilities
GET https://cyber-vision-center/api/3.0/vulnerabilities
```

**2. Syslog (CEF Format)**
```
Cyber Vision Center → Administration → Syslog
→ Add Splunk as syslog destination
→ Select event types to forward
```

CEF syslog format enables SIEM integration for:
- Security events
- Anomaly alerts
- System events

**3. Webhooks**
```
Cyber Vision Center → Integrations → Webhooks
→ Configure Splunk HEC endpoint
```

### Splunk Apps & Add-ons

| Component | Splunkbase ID | Purpose |
|-----------|---------------|---------|
| Cisco Cyber Vision Add-on | (via Catalyst Add-on) | API data collection |
| Cisco Cyber Vision App | Splunkbase | OT security dashboards |

### Sample SPL Queries

```spl
# Asset inventory by vendor
index=cybervision sourcetype="cisco:cybervision:components"
| stats count by asset_vendor, asset_model
| sort - count

# High-severity vulnerabilities
index=cybervision sourcetype="cisco:cybervision:vulnerabilities" cvss_score>=7.0
| stats count by cve_id, asset_name
| sort - cvss_score

# Anomalous communications
index=cybervision sourcetype="cisco:cybervision:events" event_type="anomaly"
| stats count by src_asset, dest_asset, protocol
| sort - count

# Zone communication matrix
index=cybervision sourcetype="cisco:cybervision:flows"
| stats count by src_zone, dest_zone
| xyseries src_zone dest_zone count
```

---

## Cisco Secure Firewall (FTD/ASA)

### Product Overview
Next-generation firewall providing intrusion prevention, malware defense, URL filtering, and application visibility.

### Use Cases

| Use Case | Description | Data Sources |
|----------|-------------|--------------|
| **Intrusion Detection** | IPS signature alerts | Intrusion events |
| **Malware Analysis** | AMP file verdicts | Malware events |
| **Connection Logging** | Traffic flow records | Connection events |
| **URL Filtering** | Web category blocking | URL events |
| **Application Control** | App identification, blocking | Application events |
| **Security Intelligence** | IP/domain reputation | SI events |
| **User Activity** | Identity-based logging | User events |

### Data Types Created

| Sourcetype | Description | Key Fields |
|------------|-------------|------------|
| `cisco:firepower:estreamer` | eStreamer events | rec_type, src_ip, dest_ip |
| `cisco:ftd:syslog` | FTD syslog messages | severity, message, interface |
| `cisco:asa:syslog` | ASA syslog messages | message_id, src, dst |
| `cisco:firepower:intrusion` | IDS/IPS events | signature, action, priority |
| `cisco:firepower:connection` | Connection logs | app, bytes, action |
| `cisco:firepower:malware` | Malware events | file_name, sha256, disposition |

### Export Methods

**1. eStreamer (Recommended for FMC)**
```
FMC → System → Integration → eStreamer
→ Create eStreamer client certificate
→ Download PKCS12 certificate
→ Configure Splunk Add-on with certificate
```

eStreamer event types:
- Intrusion events (IDS/IPS alerts)
- Connection events (flow logs)
- File events (file transfers)
- Malware events (AMP verdicts)
- Discovery events (host profiles)

**2. Syslog (FTD Direct)**
```
FMC → Devices → Platform Settings → Syslog
→ Add syslog server (Splunk IP)
→ Configure logging for access control rules
```

**3. Syslog (ASA)**
```
ASA(config)# logging host inside 10.1.1.100
ASA(config)# logging trap informational
ASA(config)# logging enable
```

### Splunk Apps & Add-ons

| Component | Splunkbase ID | Purpose |
|-----------|---------------|---------|
| Cisco Security Cloud | 7404 | Current recommended app |
| Cisco Secure Firewall App | 4388 | Dashboards (EOL) |
| Cisco eStreamer eNcore Add-on | 3662 | eStreamer data collection |

### Sample SPL Queries

```spl
# Top intrusion signatures
index=firepower sourcetype="cisco:firepower:intrusion"
| stats count by signature_name, priority
| sort - count | head 20

# Blocked connections by policy
index=firepower sourcetype="cisco:firepower:connection" action="Block"
| stats count by ac_policy, ac_rule, app_name
| sort - count

# Malware detections
index=firepower sourcetype="cisco:firepower:malware" disposition="Malware"
| stats count by file_name, sha256, src_ip
| sort - count

# Connection volume by application
index=firepower sourcetype="cisco:firepower:connection"
| stats sum(bytes_in) as bytes_in, sum(bytes_out) as bytes_out by app_name
| eval total_GB = round((bytes_in + bytes_out)/1024/1024/1024, 2)
| sort - total_GB
```

---

## Unified Cisco App: Cisco Enterprise Networking for Splunk Platform

### Overview
Single platform app that consolidates dashboards for multiple Cisco products.

### Supported Products
- Cisco Identity Services Engine (ISE)
- Cisco Catalyst SD-WAN
- Cisco Catalyst Center
- Cisco Cyber Vision
- Cisco Meraki
- Cisco ThousandEyes

### Installation Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                    SPLUNK SEARCH HEAD                          │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Cisco Enterprise Networking for Splunk Platform (7539) │  │
│  │  (Dashboards & Visualizations)                          │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
                              │
┌───────────────────────────────────────────────────────────────┐
│                    SPLUNK INDEXER                              │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Cisco Catalyst Add-on for Splunk (7538)                │  │
│  │  (Field extractions, CIM mapping)                       │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
                              │
┌───────────────────────────────────────────────────────────────┐
│              SPLUNK HEAVY FORWARDER (if needed)               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Cisco Catalyst Add-on for Splunk (7538)                │  │
│  │  (Modular inputs, API collection)                       │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

### Configuration Macros

```ini
# macros.conf
[cisco_catalyst_app_index]
definition = index IN ("main", "cisco")

[cisco_catalyst_app_sourcetypes]
definition = sourcetype IN ("cisco:ise*", "cisco:sdwan*", "cisco:dnac*", "meraki:*", "cisco:cybervision:*", "cisco:thousandeyes:*")
```

---

## Best Practices

### Index Strategy
```ini
# indexes.conf
[cisco_network]
homePath = $SPLUNK_DB/cisco_network/db
coldPath = $SPLUNK_DB/cisco_network/colddb
thawedPath = $SPLUNK_DB/cisco_network/thaweddb
maxDataSize = auto_high_volume

[cisco_security]
homePath = $SPLUNK_DB/cisco_security/db
coldPath = $SPLUNK_DB/cisco_security/colddb
thawedPath = $SPLUNK_DB/cisco_security/thaweddb
maxDataSize = auto_high_volume
```

### Data Volume Considerations

| Product | Typical Volume | Retention Recommendation |
|---------|----------------|--------------------------|
| Meraki API | Low-Medium | 90 days |
| Meraki MQTT (cameras) | High | 30 days |
| ISE Syslog | High | 90 days |
| SD-WAN NetFlow | Very High | 30 days |
| Firewall Connection Logs | Very High | 30-90 days |
| ThousandEyes Metrics | Medium | 90 days |

### CIM Compliance
All Cisco add-ons map to Splunk CIM data models:
- Network Traffic
- Authentication
- Intrusion Detection
- Malware
- Web
- Change

---

## Cisco Webex

### Product Overview
Cloud-based collaboration platform providing video conferencing (Webex Meetings), team messaging (Webex Teams), calling (Webex Calling), and contact center solutions (Webex Contact Center).

### Use Cases

| Use Case | Description | Data Sources |
|----------|-------------|--------------|
| **Meeting Analytics** | Meeting quality, participation, duration | Meeting sessions, participant data |
| **User Activity** | User engagement, adoption metrics | User activity logs |
| **Quality Monitoring** | Audio/video quality, network issues | QoS metrics |
| **Calling CDR** | Call detail records, usage patterns | Webex Calling CDR |
| **Contact Center Ops** | Agent performance, queue metrics | WxCC agent/interaction data |
| **Security Audit** | Admin actions, configuration changes | Audit logs |
| **Capacity Planning** | License usage, concurrent meetings | Licensing data |

### Data Types Created

| Sourcetype | Description | Key Fields |
|------------|-------------|------------|
| `cisco:webex:meetings:history` | Historical meeting sessions | confID, hostEmail, duration, participants |
| `cisco:webex:meetings:attendee` | Meeting participant details | email, joinTime, leaveTime, quality |
| `cisco:webex:meetings:session` | Real-time session data | meetingId, status, hostId |
| `cisco:webex:users` | User directory | email, displayName, roles, status |
| `cisco:webex:calling:cdr` | Call detail records | callingNumber, calledNumber, duration |
| `cisco:webex:audit` | Admin audit events | adminEmail, action, target, timestamp |

### Export Methods

**1. REST API (Recommended)**
```bash
# Webex API base URL
https://webexapis.com/v1/

# Authentication (OAuth 2.0 or Integration Token)
Authorization: Bearer YOUR_ACCESS_TOKEN

# Example: List meetings
GET https://webexapis.com/v1/meetings?from=2024-01-01T00:00:00Z&to=2024-01-31T23:59:59Z
```

**API Endpoints for Splunk:**
- `/meetings` - Meeting metadata
- `/meetingParticipants` - Participant details
- `/recordings` - Recording metadata
- `/people` - User directory
- `/admin/audit/events` - Audit logs (Admin API)
- `/telephony/calls/history` - Calling CDR

**2. XML API (Legacy - Webex Meetings)**
```xml
<!-- History Service for meeting data -->
POST https://site.webex.com/WBXService/XMLService

<serv:message xmlns:serv="http://www.webex.com/schemas/2002/06/service">
  <header>
    <securityContext>
      <webExID>admin@company.com</webExID>
      <password>password</password>
      <siteName>site</siteName>
    </securityContext>
  </header>
  <body>
    <bodyContent xsi:type="history:LstmeetingusageHistory">
      <startTimeScope>
        <sessionStartTimeStart>01/01/2024 00:00:00</sessionStartTimeStart>
        <sessionStartTimeEnd>01/31/2024 23:59:59</sessionStartTimeEnd>
      </startTimeScope>
    </bodyContent>
  </body>
</serv:message>
```

**3. Webhooks (Real-time Events)**
```
Webex Developer Portal → Create Webhook
→ Resource: meetings, memberships, messages
→ Event: created, updated, deleted, ended
→ Target URL: Splunk HEC endpoint
```

**4. Webex Contact Center Integration (SplunkBridge)**
```
Consilium SplunkBridge → Webex Contact Center
→ OAuth 2.0 authentication
→ Real-time agent status, interaction data
→ Push to Splunk HEC
```

### Splunk Apps & Add-ons

| Component | Splunkbase ID | Purpose |
|-----------|---------------|---------|
| Cisco Webex Meetings Add-on | 4991 | XML API data collection |
| Cisco Webex Meetings App | 4992 | Dashboards and reports |
| Cisco Webex Add-on (REST) | 5781 | REST API data collection |

### Sample SPL Queries

```spl
# Meeting duration analysis
index=webex sourcetype="cisco:webex:meetings:history"
| stats avg(duration) as avg_duration, count as total_meetings by hostEmail
| eval avg_duration_min = round(avg_duration/60, 2)
| sort - total_meetings

# Participant engagement
index=webex sourcetype="cisco:webex:meetings:attendee"
| eval attend_duration = (leaveTime - joinTime) / 60
| stats avg(attend_duration) as avg_attend, count as join_count by email
| sort - join_count

# Meeting quality issues
index=webex sourcetype="cisco:webex:meetings:attendee" 
| where audioQuality < 3 OR videoQuality < 3
| stats count by meetingId, email, audioQuality, videoQuality

# Admin audit trail
index=webex sourcetype="cisco:webex:audit"
| stats count by adminEmail, action, objectType
| sort - count
```

---

## Cisco Spaces

### Product Overview
Cloud-based indoor location services platform providing location analytics, asset tracking, customer engagement, and environmental monitoring using existing Cisco wireless infrastructure (Catalyst, Meraki, Aironet).

### Use Cases

| Use Case | Description | Data Sources |
|----------|-------------|--------------|
| **Visitor Analytics** | Foot traffic, dwell time, visit frequency | Location events |
| **Space Utilization** | Occupancy, density monitoring | Presence data |
| **Asset Tracking** | Real-time location of tagged assets | BLE tag events |
| **Customer Engagement** | Captive portal, proximity marketing | Profile, engagement events |
| **Contact Tracing** | Exposure notification, proximity history | Location history |
| **Environmental Monitoring** | Temperature, humidity, air quality | IoT sensor data |
| **Wayfinding** | Indoor navigation analytics | Path data |

### Data Types Created

| Sourcetype | Description | Key Fields |
|------------|-------------|------------|
| `cisco:spaces:location` | Device location updates | macAddress, x, y, floor, timestamp |
| `cisco:spaces:entry` | Zone entry events | deviceId, zoneId, entryTime |
| `cisco:spaces:exit` | Zone exit events | deviceId, zoneId, exitTime, dwellTime |
| `cisco:spaces:presence` | Presence detection | macAddress, apMac, rssi |
| `cisco:spaces:profile` | User profile updates | deviceId, email, attributes |
| `cisco:spaces:iot` | IoT sensor telemetry | sensorId, metric, value |
| `cisco:spaces:asset` | Asset location events | tagId, assetName, location |

### Export Methods

**1. Firehose API (Recommended - Streaming)**
```
Cisco Spaces Partner Dashboard → Create Application
→ Configure Firehose channels
→ Select event types (location, entry/exit, IoT)
→ Configure push destination (AWS SQS, Azure Event Hub, Kafka)
```

Firehose Event Types:
- `DEVICE_LOCATION_UPDATE` - Real-time location
- `DEVICE_ENTRY` - Zone entry
- `DEVICE_EXIT` - Zone exit
- `DEVICE_PRESENCE` - AP association
- `IOT_TELEMETRY` - Sensor data
- `PROFILE_UPDATE` - User profile changes

**Push Channel Configuration (AWS SQS Example):**
```json
{
  "type": "aws_sqs",
  "awsRegion": "us-east-1",
  "queueUrl": "https://sqs.us-east-1.amazonaws.com/123456789/spaces-events",
  "accessKeyId": "AKIA...",
  "secretAccessKey": "..."
}
```

**2. REST API (Location API)**
```bash
# Base URL
https://dnaspaces.io/api/location/v1/

# Authentication
Authorization: Bearer {API_TOKEN}

# Get client locations
GET https://dnaspaces.io/api/location/v1/clients

# Get location history
GET https://dnaspaces.io/api/location/v1/history?deviceId={mac}&startTime={epoch}&endTime={epoch}
```

**3. Data Export (Batch)**
```
Cisco Spaces Dashboard → Setup → Data Export
→ Configure SFTP, AWS S3, Azure Blob, or Google Cloud
→ Select data types (visits, analytics, IoT)
→ Set frequency (daily, hourly)
```

Export formats: CSV, JSON

**4. CMX On-Premises API (Legacy)**
```bash
# CMX REST API
GET https://cmx-server/api/location/v2/clients

# CMX Notification Service (push)
POST /api/config/v1/notification
{
  "name": "splunk-location",
  "rules": [{"conditions": {"condition": "movement.distance>10"}}],
  "subscribers": [{"receivers":[{"uri":"http://splunk-hec:8088/services/collector"}]}]
}
```

### Splunk Integration Pattern

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Cisco Spaces    │     │  Message Queue   │     │     Splunk       │
│  Firehose API    │────▶│  (SQS/EventHub/  │────▶│  HEC Endpoint    │
│                  │     │   Kafka)         │     │                  │
└──────────────────┘     └──────────────────┘     └──────────────────┘
         │                                                 │
         │ (Alternative: Direct REST polling)              │
         └─────────────────────────────────────────────────┘
```

Custom Integration Script Example:
```python
import requests
import json

# Firehose Pull Channel
firehose_url = "https://partners.dnaspaces.io/api/partners/v1/firehose/events"
headers = {"Authorization": f"Bearer {API_KEY}"}

response = requests.get(firehose_url, headers=headers, stream=True)
for line in response.iter_lines():
    if line:
        event = json.loads(line)
        # Forward to Splunk HEC
        requests.post(
            "https://splunk:8088/services/collector/event",
            headers={"Authorization": "Splunk HEC_TOKEN"},
            json={"event": event, "sourcetype": f"cisco:spaces:{event['eventType'].lower()}"}
        )
```

### Sample SPL Queries

```spl
# Visitor traffic by zone
index=spaces sourcetype="cisco:spaces:entry"
| timechart span=1h count by zoneName

# Average dwell time
index=spaces sourcetype="cisco:spaces:exit"
| stats avg(dwellTime) as avg_dwell_sec by zoneName
| eval avg_dwell_min = round(avg_dwell_sec/60, 2)

# Space utilization (occupancy)
index=spaces sourcetype="cisco:spaces:location"
| stats dc(macAddress) as unique_devices by floorId
| lookup floor_capacity.csv floorId OUTPUT capacity
| eval utilization_pct = round((unique_devices/capacity)*100, 1)

# Asset location tracking
index=spaces sourcetype="cisco:spaces:asset"
| stats latest(x) as x, latest(y) as y, latest(floor) as floor by tagId, assetName
| table assetName, floor, x, y

# IoT sensor analytics
index=spaces sourcetype="cisco:spaces:iot" metric="temperature"
| timechart span=15m avg(value) as avg_temp by sensorId
```

---

## Cisco Edge Intelligence

### Product Overview
IoT data orchestration software that extracts, transforms, governs, and delivers connected asset data from the IoT edge to cloud destinations. Runs on Cisco industrial routers (IR1101, IR829) and compute gateways (IC3000).

### Use Cases

| Use Case | Description | Data Sources |
|----------|-------------|--------------|
| **Industrial Monitoring** | PLC, SCADA, sensor data collection | OPC-UA, Modbus |
| **Predictive Maintenance** | Equipment telemetry for ML models | Sensor streams |
| **Process Optimization** | Real-time production metrics | Process data |
| **Energy Management** | Power consumption monitoring | Energy meters |
| **Quality Control** | Production quality metrics | Inspection systems |
| **Fleet Management** | Vehicle/equipment telemetry | GPS, telematics |
| **Environmental Monitoring** | Temperature, pressure, flow | Industrial sensors |

### Data Types Created

| Sourcetype | Description | Key Fields |
|------------|-------------|------------|
| `cisco:ei:telemetry` | Sensor/PLC telemetry | assetId, metric, value, timestamp |
| `cisco:ei:event` | Discrete events | eventType, assetId, payload |
| `cisco:ei:alarm` | Threshold alarms | alarmId, severity, condition |
| `cisco:ei:opcua` | OPC-UA node values | nodeId, value, quality, timestamp |
| `cisco:ei:modbus` | Modbus register values | address, value, slaveId |
| `cisco:ei:mqtt` | MQTT message payloads | topic, payload |

### Export Methods

**1. Native Splunk Destination (Recommended)**

Edge Intelligence has built-in Splunk support via HTTP Event Collector:

```
EI Local Manager → Pipelines → Add Destination → Splunk
→ Configure HEC URL and token
→ Select single or batch payload mode
```

**Splunk Destination Configuration:**
```json
{
  "destinationType": "splunk",
  "name": "splunk-production",
  "config": {
    "url": "https://splunk-hec.company.com:8088/services/collector/event",
    "token": "your-hec-token",
    "index": "cisco_ei",
    "sourcetype": "cisco:ei:telemetry",
    "batchSize": 100,
    "batchTimeout": 5000
  }
}
```

**2. Pipeline Configuration**

Data flow: Source → Transform → Destination

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Data Sources   │     │  Edge Transform │     │   Destinations  │
├─────────────────┤     ├─────────────────┤     ├─────────────────┤
│  • OPC-UA       │────▶│  • Data Rules   │────▶│  • Splunk HEC   │
│  • Modbus TCP   │     │  • Data Logic   │     │  • MQTT Broker  │
│  • Modbus Serial│     │    (JavaScript) │     │  • AWS IoT      │
│  • MQTT         │     │  • Filtering    │     │  • Azure IoT    │
│  • EIP/CIP      │     │  • Aggregation  │     │                 │
│  • Serial       │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

**3. Supported Industrial Protocols**

| Protocol | Connection Type | Use Case |
|----------|----------------|----------|
| OPC-UA | TCP/IP | PLCs, SCADA, historians |
| Modbus TCP | TCP/IP | Industrial controllers |
| Modbus RTU | Serial | Legacy equipment |
| MQTT | TCP/IP | IoT sensors, gateways |
| EIP/CIP | TCP/IP | Allen-Bradley PLCs |
| Serial | RS-232/485 | Legacy devices |
| NTCIP | TCP/IP | Traffic systems |

**4. Data Logic (Edge Transform)**

JavaScript-based transformation at the edge:

```javascript
// Example: Calculate running average and send to Splunk
var samples = [];
var windowSize = 10;

function onData(data) {
    samples.push(data.value);
    if (samples.length > windowSize) {
        samples.shift();
    }
    
    var avg = samples.reduce((a, b) => a + b, 0) / samples.length;
    
    // Only publish if significant change
    if (Math.abs(avg - lastAvg) > threshold) {
        publish({
            metric: data.metric + "_avg",
            value: avg,
            timestamp: Date.now(),
            assetId: data.assetId
        });
        lastAvg = avg;
    }
}
```

### Splunk Integration Architecture

```
Industrial Network                Edge                    Cloud/Enterprise
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  PLCs           │     │  Cisco IR1101    │     │                  │
│  Sensors        │────▶│  + Edge Intel    │────▶│  Splunk Platform │
│  SCADA          │     │  Agent           │     │                  │
│  Historians     │     │                  │     │  • Indexing      │
└─────────────────┘     │  - Protocol      │     │  • Analytics     │
                        │    Conversion    │     │  • Dashboards    │
                        │  - Edge Compute  │     │  • Alerting      │
                        │  - Filtering     │     │                  │
                        └──────────────────┘     └──────────────────┘
```

### Sample SPL Queries

```spl
# Equipment telemetry overview
index=cisco_ei sourcetype="cisco:ei:telemetry"
| stats latest(value) as current_value, avg(value) as avg_value by assetId, metric
| table assetId, metric, current_value, avg_value

# OPC-UA node monitoring
index=cisco_ei sourcetype="cisco:ei:opcua"
| timechart span=1m avg(value) by nodeId

# Modbus register trends
index=cisco_ei sourcetype="cisco:ei:modbus"
| where address >= 40001 AND address <= 40010
| timechart span=5m avg(value) by address

# Alarm analysis
index=cisco_ei sourcetype="cisco:ei:alarm"
| stats count by assetId, alarmId, severity
| sort - severity, - count

# Data throughput monitoring
index=cisco_ei
| timechart span=1h count by sourcetype

# Edge device health
index=cisco_ei sourcetype="cisco:ei:telemetry" metric="cpu_usage" OR metric="memory_usage"
| stats latest(value) as current by host, metric
| xyseries host metric current
```

### Deployment Platforms

| Device | Description | Typical Use |
|--------|-------------|-------------|
| Cisco IR1101 | Compact industrial router | Remote sites, pump stations |
| Cisco IR829 | Rugged industrial router | Harsh environments |
| Cisco IC3000 | Industrial compute gateway | Heavy processing needs |
| Cisco Catalyst IE | Industrial Ethernet switches | With IOx capability |

### Best Practices

1. **Edge Filtering**: Reduce bandwidth by filtering data at the edge
   - Send only on value change
   - Use deadband thresholds
   - Aggregate high-frequency data

2. **Batch vs. Single Payload**:
   - Batch mode: Better for high-volume, reduces HTTP overhead
   - Single mode: Lower latency for critical alerts

3. **Data Governance**:
   - Use data policies to control data routing
   - Implement role-based access to pipelines
   - Version control pipeline configurations

---

## Additional Data Volume Considerations

| Product | Typical Volume | Retention Recommendation |
|---------|----------------|--------------------------|
| Webex Meetings | Low-Medium | 90 days |
| Webex CDR | Medium | 180 days |
| Cisco Spaces Location | High | 30-90 days |
| Cisco Spaces IoT | Medium-High | 90 days |
| Edge Intelligence | Variable (edge-filtered) | 90 days |
