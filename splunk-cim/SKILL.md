---
name: splunk-cim
description: "Splunk Common Information Model (CIM) reference for data normalization. Use when: (1) Mapping source data to CIM data models (Network Traffic, Authentication, Alerts, Change, Endpoint, Performance), (2) Creating props.conf field aliases and calculated fields for CIM compliance, (3) Configuring eventtypes.conf and tags.conf for data model routing, (4) Writing tstats queries against accelerated CIM data models, (5) Validating CIM compliance with SA-cim_vladiator, (6) Understanding CIM vs Operational Telemetry data model selection for OT/IoT use cases."
---

# Splunk Common Information Model (CIM) Reference

## Overview

The Splunk Common Information Model (CIM) is a search-time schema delivered as an add-on (`Splunk_SA_CIM`) containing **26 preconfigured data models** that normalize field names and event tags across disparate data sources. CIM acts as the foundational normalization layer for Splunk Enterprise Security (ES), enabling correlation searches, accelerated dashboards, and cross-vendor analytics without modifying raw data.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CIM ARCHITECTURE                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  RAW DATA              NORMALIZATION              DATA MODELS               │
│  ─────────             ─────────────              ───────────               │
│                                                                              │
│  ┌─────────────┐     ┌──────────────────┐     ┌──────────────────┐         │
│  │ Firewall    │     │ props.conf       │     │                  │         │
│  │ Logs        │────▶│ • Field aliases  │────▶│  Network_Traffic │         │
│  └─────────────┘     │ • Calc fields    │     │  Authentication  │         │
│                      └──────────────────┘     │  Alerts          │         │
│  ┌─────────────┐            │                 │  Change          │         │
│  │ Auth        │            │                 │  Endpoint        │         │
│  │ Events      │────────────┤                 │  Performance     │         │
│  └─────────────┘            │                 │  ...             │         │
│                      ┌──────────────────┐     └──────────────────┘         │
│  ┌─────────────┐     │ eventtypes.conf  │            │                     │
│  │ IDS/IPS     │────▶│ tags.conf        │────────────┘                     │
│  │ Alerts      │     │ • Route to model │                                  │
│  └─────────────┘     └──────────────────┘                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Complete Inventory of CIM Data Models

The CIM add-on ships with 26 data models (24 active, 2 deprecated). Each model defines a domain with standardized field names, tag-based constraints, and hierarchical object structure.

| # | Data Model | Status | Domain | Required Tags |
|---|-----------|--------|--------|---------------|
| 1 | **Alerts** | Active | Security/equipment alerting | `alert` |
| 2 | Application State | Deprecated | Replaced by Endpoint | — |
| 3 | **Authentication** | Active | Login/authentication events | `authentication` |
| 4 | **Certificates** | Active | SSL/TLS certificate data | `certificate`, `ssl` |
| 5 | **Change** | Active | CRUD on resources | `change` |
| 6 | Change Analysis | Deprecated | Replaced by Change | — |
| 7 | **Data Access** | Active | Data access events | `data`, `access` |
| 8 | **Databases** | Active | Database activities | `database` |
| 9 | **Data Loss Prevention** | Active | DLP events | `dlp`, `incident` |
| 10 | **Email** | Active | Email activity | `email` |
| 11 | **Endpoint** | Active | Process/service/file/port | Varies by dataset |
| 12 | **Event Signatures** | Active | Signature-based events | `track_event_signatures` |
| 13 | **Interprocess Messaging** | Active | Message queue events | `messaging` |
| 14 | **Intrusion Detection** | Active | IDS/IPS events | `ids`, `attack` |
| 15 | **Inventory** | Active | Asset inventory | `inventory` |
| 16 | **JVM** | Active | Java Virtual Machine metrics | `jvm` |
| 17 | **Malware** | Active | Malware detection | `malware`, `attack` |
| 18 | **Network Resolution (DNS)** | Active | DNS resolution events | `network`, `resolution`, `dns` |
| 19 | **Network Sessions** | Active | Network session data | `network`, `session` |
| 20 | **Network Traffic** | Active | Network flow data | `network`, `communicate` |
| 21 | **Performance** | Active | System performance metrics | `performance` + subcategory |
| 22 | **Splunk Audit Logs** | Active | Splunk internal audit | `splunk_audit` |
| 23 | **Ticket Management** | Active | Ticketing system events | `ticketing` |
| 24 | **Updates** | Active | Software/system updates | `update` |
| 25 | **Vulnerabilities** | Active | Vulnerability scan data | `vulnerability`, `report` |
| 26 | **Web** | Active | Web/HTTP activity | `web` |

---

## Object Hierarchy and Inheritance

Every CIM data model follows a parent-child hierarchy. The root/base dataset contains the broadest set of fields and tag constraints. Child datasets inherit **all** parent fields and add narrower constraints.

**Authentication hierarchy example:**
```
Authentication (tag: authentication)
├── Default_Authentication (tags: authentication, default)
├── Insecure_Authentication (tags: authentication, cleartext OR insecure)
└── Privileged_Authentication (tags: authentication, privileged)
```

**Change hierarchy:**
```
All_Changes (tag: change)
├── Auditing_Changes (tags: change, audit)
├── Endpoint_Changes (tags: change, endpoint)
├── Network_Changes (tags: change, network)
├── Account_Management (tags: change, account)
└── Instance_Changes (tags: change, instance)
```

**Endpoint hierarchy** (unique architecture):
```
Endpoint (not directly searchable)
├── Ports (tags: listening, port)
├── Processes (tags: process, report)
├── Services (tags: service, report)
├── Filesystem (tags: endpoint, filesystem)
└── Registry (tags: endpoint, registry)
```

Tags are **cumulative** up the hierarchy. A child dataset requires its own tags plus all parent tags.

---

## Field Naming Conventions

CIM uses consistent field naming patterns across all models:

**Source/Destination Pattern:**
- `src` and `dest` identify endpoints of an interaction
- These alias from `src_ip`, `src_host`, `src_nt_host` and `dest_` equivalents
- `dvc` identifies the intermediary reporting device

**User Fields:**
- `user` is the primary actor
- `src_user` handles privilege escalation scenarios
- Fields suffixed with `_bunit`, `_category`, `_priority` are auto-populated by ES

**Action/Status Pattern:**
- `action` holds normalized values: `allowed`, `blocked`, `success`, `failure`, `created`, `deleted`, `modified`
- `status` reflects outcome
- `signature` and `signature_id` provide human-readable and machine-parseable event identification

---

## CIM Implementation Files

CIM compliance is achieved through four configuration files. The **search-time evaluation order** is:
1. Field extraction
2. Field aliasing
3. Calculated fields (EVAL)
4. Lookups
5. Event type matching
6. Tag assignment
7. CIM data model constraint matching

### props.conf - Field Mapping

**Field Aliases** map vendor-specific names to CIM-standard names:

```ini
[acme:firewall]
# Network Traffic mappings
FIELDALIAS-src       = src_addr AS src
FIELDALIAS-dest      = dst_addr AS dest
FIELDALIAS-dest_ip   = dst_addr AS dest_ip
FIELDALIAS-src_port  = src_prt AS src_port
FIELDALIAS-dest_port = dst_prt AS dest_port
FIELDALIAS-transport = proto AS transport
FIELDALIAS-rule      = rule_name AS rule
FIELDALIAS-bytes_in  = bytes_rcvd AS bytes_in
FIELDALIAS-bytes_out = bytes_sent AS bytes_out
```

**ASNEW variant** (Splunk 7.2.4+) prevents overwriting existing field values:
```ini
FIELDALIAS-dest = dst ASNEW dest
```

**Calculated Fields** handle value transformation:

```ini
[acme:firewall]
# Normalize action to CIM-prescribed values
EVAL-action = case(
    fw_action=="PERMIT", "allowed",
    fw_action=="DENY", "blocked",
    fw_action=="DROP", "blocked",
    fw_action=="REJECT", "blocked",
    true(), lower(fw_action)
)

# Calculate total bytes
EVAL-bytes = bytes_sent + bytes_rcvd

# Vendor/product identification
EVAL-vendor_product = "Acme Corp Acme Firewall"

# Multi-source user field coalescing
EVAL-user = coalesce(username, src_user, source_user)
```

**Automatic Lookups** enrich events:

```ini
[acme:firewall]
LOOKUP-action_lookup = cim_action_lookup vendor_action OUTPUTNEW action
LOOKUP-protocol_lookup = acme_protocol_lookup dest_port OUTPUTNEW app
```

### transforms.conf - Lookup Definitions

```ini
[cim_action_lookup]
filename = cim_actions.csv

[acme_protocol_lookup]
filename = acme_protocol.csv
default_match = unknown
```

### eventtypes.conf - Event Categorization

Event types categorize events using saved search strings. **No pipe operators or sub-searches allowed.**

```ini
[acme_firewall_traffic]
search = sourcetype="acme:firewall"

[acme_firewall_traffic_allowed]
search = sourcetype="acme:firewall" fw_action="PERMIT"

[acme_firewall_traffic_blocked]
search = sourcetype="acme:firewall" (fw_action="DENY" OR fw_action="DROP")

[acme_auth_success]
search = sourcetype="acme:auth" result="success"

[acme_auth_failure]
search = sourcetype="acme:auth" result="failure"
```

### tags.conf - CIM Data Model Routing

Tags connect event types to CIM data model constraints. **This is the critical link:**

```ini
# Maps to Network Traffic data model (requires: network + communicate)
[eventtype=acme_firewall_traffic]
network = enabled
communicate = enabled

# Maps to Change data model (requires: change)
[eventtype=acme_config_change]
change = enabled

# Maps to Authentication data model (requires: authentication)
[eventtype=acme_auth_success]
authentication = enabled

[eventtype=acme_auth_failure]
authentication = enabled
```

**Real-world example from Palo Alto Networks TA:**

```ini
[eventtype=pan_traffic]
network = enabled
communicate = enabled

[eventtype=pan_threat]
ids = enabled
attack = enabled

[eventtype=pan_url]
web = enabled
proxy = enabled

[eventtype=pan_config]
change = enabled

[eventtype=pan_wildfire_malicious]
malware = enabled
attack = enabled
```

---

## Key Data Models for OT/IoT

### Network Traffic Data Model

Maps industrial network flows—Modbus/TCP, DNP3, EtherNet/IP, OPC UA—through the `All_Traffic` base dataset.

**Required Tags:** `network`, `communicate`

| Field | Type | Classification | Description |
|-------|------|----------------|-------------|
| **`action`** | string | **Required** | `allowed`, `blocked`, `dropped`, `teardown` |
| **`src`** | string | **Required** | Source (aliased from src_host, src_ip) |
| **`dest`** | string | **Required** | Destination (aliased from dest_host, dest_ip) |
| **`transport`** | string | **Required** | Layer 4 protocol: tcp, udp, icmp |
| `dest_port` | number | Recommended | Destination port |
| `src_port` | number | Recommended | Source port |
| `bytes` | number | Recommended | Total bytes (bytes_in + bytes_out) |
| `bytes_in` | number | Recommended | Inbound bytes |
| `bytes_out` | number | Recommended | Outbound bytes |
| `duration` | number | Recommended | Connection duration (seconds) |
| `dvc` | string | Recommended | Reporting device |
| `app` | string | Recommended | Application protocol |
| `vendor_product` | string | Recommended | Vendor and product identifier |

**Example props.conf for OT firewall:**
```ini
[ot:firewall]
FIELDALIAS-src = source_ip AS src
FIELDALIAS-dest = destination_ip AS dest
FIELDALIAS-src_port = sport AS src_port
FIELDALIAS-dest_port = dport AS dest_port
FIELDALIAS-transport = protocol AS transport
EVAL-action = case(action_code==1, "allowed", action_code==0, "blocked", true(), "unknown")
EVAL-bytes = bytes_in + bytes_out
EVAL-vendor_product = "OT Vendor Industrial Firewall"
```

### Authentication Data Model

Tracks authentication to OT systems, engineering workstations, SCADA servers.

**Required Tags:** `authentication`

| Field | Type | Classification | Description |
|-------|------|----------------|-------------|
| **`action`** | string | **Required** | `success`, `failure`, `pending`, `error` |
| **`app`** | string | **Required** | Application (ssh, rdp, scada_hmi) |
| **`user`** | string | **Required** | User logging in |
| `dest` | string | Recommended | Target host |
| `src` | string | Recommended | Source host |
| `src_user` | string | Recommended | User who initiated privilege escalation |
| `authentication_method` | string | Optional | Method (password, certificate, token) |
| `signature` | string | Optional | Event description |

**Example configuration:**
```ini
[ot:auth]
FIELDALIAS-user = username AS user
FIELDALIAS-dest = target_host AS dest
FIELDALIAS-src = source_host AS src
EVAL-action = if(status=="OK", "success", "failure")
EVAL-app = "scada_hmi"
```

### Alerts Data Model

Maps directly to ICS/SCADA alerting, OT security platform alerts, safety system notifications.

**Required Tags:** `alert`

| Field | Type | Classification | Description |
|-------|------|----------------|-------------|
| **`app`** | string | **Required** | Generating system (Nagios, Claroty, Nozomi) |
| **`dest`** | string | **Required** | Alert target |
| **`id`** | string | **Required** | Unique alert identifier |
| **`severity`** | string | **Required** | `critical`, `high`, `medium`, `low`, `informational` |
| **`type`** | string | **Required** | `alarm`, `alert`, `event`, `task`, `warning` |
| **`body`** | string | **Required** | Message body |
| `src` | string | Recommended | Alert source |
| `signature` | string | Recommended | Alert signature |

**Example configuration:**
```ini
[ot:alerts]
FIELDALIAS-id = alert_id AS id
FIELDALIAS-dest = affected_asset AS dest
EVAL-severity = case(
    priority==1, "critical",
    priority==2, "high",
    priority==3, "medium",
    priority==4, "low",
    true(), "informational"
)
EVAL-type = "alert"
EVAL-app = "ot_monitoring_system"
```

### Change Data Model

Critical for tracking PLC program uploads, firmware updates, HMI configuration changes.

**Required Tags:** `change`

| Field | Type | Classification | Description |
|-------|------|----------------|-------------|
| **`action`** | string | **Required** | `created`, `deleted`, `modified`, `started`, `stopped` |
| **`change_type`** | string | **Required** | Type (filesystem, AAA, firmware, config) |
| **`command`** | string | **Required** | Initiating command |
| **`dest`** | string | **Required** | Resource where change occurred |
| **`object`** | string | **Required** | Affected object name |
| **`object_category`** | string | **Required** | Object class (file, firmware, PLC_program) |
| **`status`** | string | **Required** | `success`, `failure` |
| **`user`** | string | **Required** | Entity performing the change |

### Performance Data Model

Directly applicable to monitoring OT infrastructure CPU, memory, storage, and facilities (temperature, power).

**Required Tags:** `performance` + subcategory tag

| Dataset | Tags | Key Fields |
|---------|------|------------|
| **CPU** | `performance`, `cpu` | `cpu_load_percent`, `cpu_load_mhz` |
| **Facilities** | `performance`, `facilities` | `temperature`, `power`, `fan_speed` |
| **Memory** | `performance`, `memory` | `mem`, `mem_free`, `mem_used` |
| **Storage** | `performance`, `storage` | `storage`, `storage_free`, `storage_used_percent` |
| **Network** | `performance`, `network` | `thruput`, `thruput_max` |
| **Uptime** | `performance`, `os`, `uptime` | `uptime` (seconds) |

### Endpoint Data Model

Maps to PLCs, HMIs, and engineering workstations. Unique architecture with directly searchable datasets.

**Processes dataset** (tags: `process`, `report`):
- Required: `action`, `dest`, `process`, `process_exec`, `process_id`, `process_name`, `process_path`, `parent_process_id`, `user`

**Services dataset** (tags: `service`, `report`):
- Required: `dest`, `service`, `service_name`, `service_path`, `start_mode`, `status`, `user`

---

## CIM Validation

### Built-in Validation Methods

The **CIM Validation (S.o.S.) data model** ships with the CIM add-on. Access via Settings → Data Models → CIM Validation (S.o.S.) → Pivot. **Never accelerate this validation data model.**

**Direct search validation:**
```spl
| datamodel Network_Traffic All_Traffic search
| search sourcetype=acme:firewall
| table src, dest, src_port, dest_port, action, transport, bytes_in, bytes_out
| fieldsummary
```

**Explore data model structure:**
```spl
| datamodelsimple type=attributes datamodel=Network_Traffic object=All_Traffic
```

### SA-cim_vladiator App

Available on Splunkbase (app/2968), this app:
- Identifies missing required fields
- Validates field values against regex patterns
- Provides rapid prototyping for CIM compliance

### tstats Queries for Accelerated Data Models

The `tstats` command queries accelerated data model summaries for **orders-of-magnitude faster** performance:

```spl
# Network Traffic: count by source and destination
| tstats summariesonly=t count
    from datamodel=Network_Traffic.All_Traffic
    where All_Traffic.action=allowed
    by All_Traffic.src, All_Traffic.dest, All_Traffic.dest_port
| sort -count

# Authentication: failed logins
| tstats summariesonly=t count
    from datamodel=Authentication
    where Authentication.action="failure"
    by Authentication.user, Authentication.src, Authentication.dest, Authentication.app
| sort -count

# Endpoint: process execution
| tstats summariesonly=t values(Processes.process)
    from datamodel=Endpoint.Processes
    groupby Processes.process_current_directory

# Audit which indexes/sourcetypes feed a data model
| tstats summariesonly=t count FROM datamodel=Network_Traffic BY index, sourcetype
```

### Common Compliance Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Tags not applied | App-level permissions | Set tags to "All Apps" scope |
| Missing field extractions | Fields not aliased to CIM names | Add FIELDALIAS in props.conf |
| Data not in models | Missing eventtype or tag chain | Verify eventtype → tag → model chain |
| Overly broad scanning | `cim_<model>_indexes` set to `(*)` | Restrict macro to relevant indexes |

---

## Data Model Acceleration

### datamodels.conf Configuration

```ini
[Network_Traffic]
acceleration                    = true
acceleration.earliest_time      = -1mon
acceleration.cron_schedule      = */5 * * * *
acceleration.backfill_time      = -7d
acceleration.max_time           = 3600
acceleration.max_concurrent     = 3
```

All CIM data models ship with acceleration **disabled by default**. Storage estimation: **accelerated DMA storage per year ≈ daily data volume × 3.4**.

### Performance Tuning

**Most impactful optimization:** Restrict `cim_<datamodel>_indexes` macros:

```
# Before (scans all indexes)
cim_Network_Traffic_indexes = (*)

# After (scans only relevant indexes)
cim_Network_Traffic_indexes = (index=firewall OR index=network)
cim_Authentication_indexes = (index=auth OR index=windows)
```

**Monitor acceleration health:**
```spl
index=_internal sourcetype=scheduler component=SavedSplunker ACCELERATE NOT skipped run_time=*
| rex field=savedsearch_id "ACCELERATE_(?:[A-F0-9\-]{36}_)?(?<acceleration>.*?)_ACCELERATE"
| timechart span=5m max(run_time) AS run_time by acceleration
```

Each DMA job should complete in **under 2 minutes**.

---

## CIM vs Operational Telemetry

### When to Use CIM

- Security events (alerts, logins, network flows)
- Log-based event data
- Enterprise Security integration
- Cross-vendor correlation
- Compliance reporting (PCI, SOC2)

### When to Use Operational Telemetry

- High-frequency sensor telemetry (temperature, pressure, vibration)
- Numeric time-series data
- OT-specific asset context
- Industrial protocol metadata
- Equipment health monitoring

### How They Complement Each Other

Both share **ES Asset Framework identity fields** (ip, mac, dns, nt_host, location), enabling joins between security events and operational telemetry on the same asset. A CIM-detected intrusion alert on a PLC can be correlated with that PLC's operational readings from the OT data model.

| Use Case | Approach | Query Method |
|----------|----------|-------------|
| Security events | CIM data models | `tstats` |
| Sensor/process telemetry | OT data model | `tstats`, `| from datamodel` |
| High-frequency numeric KPIs | Metric indexes | `mstats` |
| OT asset inventory | OT Asset model | ES Asset Framework |

---

## Complete CIM Mapping Example

### Scenario: Industrial Firewall to Network Traffic

**Source data:**
```
timestamp=2026-02-23T10:15:30Z src_addr=192.168.1.100 dst_addr=10.0.0.50 proto=tcp sport=45123 dport=502 action_code=1 bytes_sent=1024 bytes_recv=2048 rule="Allow_Modbus"
```

**props.conf:**
```ini
[industrial:firewall]
TIME_FORMAT = %Y-%m-%dT%H:%M:%SZ
TIME_PREFIX = timestamp=
SHOULD_LINEMERGE = false
KV_MODE = auto

# CIM Field Aliases
FIELDALIAS-src = src_addr AS src
FIELDALIAS-src_ip = src_addr AS src_ip
FIELDALIAS-dest = dst_addr AS dest
FIELDALIAS-dest_ip = dst_addr AS dest_ip
FIELDALIAS-src_port = sport AS src_port
FIELDALIAS-dest_port = dport AS dest_port
FIELDALIAS-transport = proto AS transport
FIELDALIAS-rule = rule AS rule

# CIM Calculated Fields
EVAL-action = case(action_code==1, "allowed", action_code==0, "blocked", true(), "unknown")
EVAL-bytes = bytes_sent + bytes_recv
EVAL-bytes_in = bytes_recv
EVAL-bytes_out = bytes_sent
EVAL-vendor_product = "Industrial Vendor Industrial Firewall"
EVAL-app = case(dport==502, "modbus", dport==44818, "ethernetip", dport==4840, "opcua", true(), "unknown")
```

**eventtypes.conf:**
```ini
[industrial_firewall_traffic]
search = sourcetype="industrial:firewall"

[industrial_firewall_allowed]
search = sourcetype="industrial:firewall" action_code=1

[industrial_firewall_blocked]
search = sourcetype="industrial:firewall" action_code=0
```

**tags.conf:**
```ini
[eventtype=industrial_firewall_traffic]
network = enabled
communicate = enabled

[eventtype=industrial_firewall_allowed]
network = enabled
communicate = enabled

[eventtype=industrial_firewall_blocked]
network = enabled
communicate = enabled
```

**Validation query:**
```spl
| datamodel Network_Traffic All_Traffic search
| search sourcetype="industrial:firewall"
| table _time, src, dest, src_port, dest_port, transport, action, bytes, app, vendor_product
| head 10
```

---

## References

- [Splunk CIM Documentation](https://docs.splunk.com/Documentation/CIM/latest/User/Overview)
- [CIM Add-on on Splunkbase](https://splunkbase.splunk.com/app/1621)
- [SA-cim_vladiator](https://splunkbase.splunk.com/app/2968)
- [Data Model Reference Tables](https://docs.splunk.com/Documentation/CIM/latest/User/Howtousethesereferencetables)

---

## Related Skills

- **splunk-admin**: Core Splunk configuration, SPL, and knowledge objects
- **splunk-edge-hub**: Edge Hub data collection and OT protocols
- **cisco-edge-intelligence**: Cisco EI deployment and HEC integration
- **splunk-operational-telemetry**: OT-specific data model (planned)
