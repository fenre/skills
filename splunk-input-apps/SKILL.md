---
name: splunk-input-apps
description: >-
  Select the optimal Splunk ingest method, Technology Add-on (TA), or input app
  for any data source when researching or building a data pipeline. Covers app
  types (TAs, apps, SAs, DAs), collection methods (UF, HF, HEC, SC4S, DB Connect,
  modular inputs), Splunkbase selection, CIM alignment, and deployment roles.
  Use when choosing how to onboard a new data source into Splunk.
---

# Splunk Ingest Apps & Data Pipeline Selection Guide

> **Purpose**: Provide authoritative guidance on selecting the optimal Splunk ingest method, Technology Add-on (TA), or input app for any given data source when researching and developing a data pipeline.
>
> **Sources**: Splunkbase (splunkbase.splunk.com), Splunk Lantern (lantern.splunk.com), Splunk Documentation (docs.splunk.com / help.splunk.com)

---

## 1. Core Concepts: App Types & Their Roles

### 1.1 Technology Add-ons (TAs)
- **Naming convention**: `Splunk_TA_<vendor>` or `TA-<vendor>`
- **Primary function**: Data collection (inputs) + field extraction + CIM normalization
- **Contains**: `inputs.conf`, `props.conf`, `transforms.conf`, modular inputs, scripted inputs, eventtypes, tags, lookups
- **Does NOT contain**: Dashboards or user-facing views (usually)
- **Deployment**: Install on forwarders (for inputs), indexers (for index-time parsing), and search heads (for search-time extractions and CIM mapping)
- **CIM compliance**: Splunk-built TAs map to Common Information Model data models, enabling compatibility with ES, ITSI, and CIM-based apps

### 1.2 Apps (Visualization/Workflow)
- **Naming convention**: `SplunkAppFor<Product>` or descriptive names
- **Primary function**: Dashboards, saved searches, reports, workflow actions
- **Relationship to TAs**: Apps depend on TAs for data collection and field normalization; apps consume the data that TAs prepare
- **Deployment**: Search heads only

### 1.3 Support Add-ons (SAs) and Domain Add-ons (DAs)
- **SA**: Provide shared saved searches, macros, or lookups used across multiple apps
- **DA**: Provide domain-specific knowledge objects (e.g., CIM data models)
- **Example**: `SA-CIM` (Splunk Common Information Model Add-on)

---

## 2. Data Collection Methods (Ingest Mechanisms)

When recommending an ingest approach, always evaluate the data source against these primary collection methods. The choice of method is often dictated by the data source type, not just preference.

### 2.1 Universal Forwarder (UF)
- **Best for**: File/directory monitoring, Windows event logs, *nix logs, scripted inputs using OS-native tools
- **How it works**: Lightweight agent installed on the source host; monitors files/directories, Windows inputs, or runs scripted inputs; forwards raw (unparsed) data to indexers via S2S protocol
- **Strengths**: Minimal footprint (~50MB RAM), lossless delivery with checkpointing, built-in load balancing, SSL/TLS support, scales to tens of thousands of endpoints
- **Limitations**: Cannot run modular inputs requiring Python (use HF instead), cannot parse/transform data (parsing happens on indexers), cannot run apps like DBConnect
- **Splunk Cloud consideration**: Requires Splunk Cloud credentials package for secure connectivity
- **When to use**: Default choice for any host-based log collection. Always prefer UF over HF unless functionality demands otherwise

### 2.2 Heavy Forwarder (HF)
- **Best for**: API-based collection (modular inputs), database connectivity (DBConnect), complex routing/filtering, protocol translation
- **How it works**: Full Splunk Enterprise instance with indexing disabled; runs the complete parsing pipeline; supports all apps and modular inputs
- **Strengths**: Can run any Splunk app/TA with modular inputs, can parse and route data pre-indexing, supports complex transformations
- **Limitations**: Higher resource footprint (4x+ throughput overhead vs UF), sends parsed/cooked data (up to 6x more network traffic than UF), introduces architectural complexity
- **Key use cases requiring HF**:
  - Splunk DB Connect (RDBMS ingestion)
  - Checkpoint OPSEC LEA
  - Cisco IPS (legacy)
  - VMware monitoring via modular input
  - Any TA with modular inputs requiring a full Python stack
- **When to use**: Only when the TA or data source requires it. Never use HF where UF suffices

### 2.3 HTTP Event Collector (HEC)
- **Best for**: Application telemetry, IoT devices, serverless/cloud functions, any HTTP-capable source, container environments
- **How it works**: Token-based JSON/raw API endpoint on indexers or intermediate tier; data pushed via HTTP/HTTPS POST; supports `/event` (structured JSON) and `/raw` (raw text) endpoints
- **Strengths**: No forwarder installation needed (agentless), token-based auth (no stored credentials), supports GZIP compression and batch processing, highly scalable with load balancers, excellent indexer distribution
- **Limitations**: Push-only (source must be able to make HTTP calls), timestamp handling differs between `/event` and `/raw` endpoints
- **Key use cases**:
  - Application logging (custom apps, microservices)
  - Cloud functions (Azure Functions, AWS Lambda, GCP Cloud Functions)
  - IoT/OT devices with HTTP capability
  - Splunk Connect for Syslog (SC4S) backend
  - Splunk OpenTelemetry Collector backend
  - Kubernetes log collection
- **Splunk Cloud consideration**: HEC is natively available on Splunk Cloud indexers; can also be fronted by intermediate HFs for additional processing

### 2.4 Syslog Collection
Two primary approaches exist — always prefer SC4S for new deployments:

#### 2.4.1 Splunk Connect for Syslog (SC4S) — **Recommended**
- **Best for**: Any syslog-emitting device (firewalls, network gear, security appliances, Linux hosts)
- **How it works**: Containerized syslog-ng distribution; receives syslog (UDP/TCP/TLS/RELP); auto-identifies vendor/product; sets sourcetype and metadata; forwards to Splunk via HEC
- **Strengths**: Out-of-box source identification for 100+ vendors, better indexer load balancing than file-based syslog, containerized deployment (Podman/Docker), TLS support, community-maintained parser library
- **SC4S Lite**: Lightweight variant with pluggable modular parsers for performance-critical deployments
- **When to use**: All new syslog deployments. Migrate existing UF-based syslog when scaling or redesigning

#### 2.4.2 UF-Based Syslog (Legacy)
- **How it works**: rsyslog/syslog-ng writes to files → UF monitors files → forwards to indexers
- **When to use**: Only if SC4S cannot be deployed (e.g., no container runtime available, existing stable deployment with no scaling needs)

### 2.5 Modular Inputs (API-Based Collection)
- **Best for**: Cloud services, SaaS platforms, any vendor API
- **How it works**: Python-based scripts packaged within TAs; run on HF or IDM (Inputs Data Manager); poll vendor APIs on configurable intervals
- **Strengths**: Purpose-built for specific vendor APIs, handle authentication/pagination/rate-limiting, checkpoint-based for incremental collection
- **Runs on**: Heavy Forwarder (Enterprise), Inputs Data Manager (Splunk Cloud classic), Search Head / SHC (Splunk Cloud Victoria)
- **Examples**: AWS TA, Azure TA, GCP TA, O365 TA, CrowdStrike TA, Okta TA, ServiceNow TA

### 2.6 Splunk DB Connect
- **Best for**: RDBMS/database ingestion (SQL Server, Oracle, MySQL, PostgreSQL, etc.)
- **How it works**: JDBC-based connectivity; runs on HF; supports scheduled queries, rising column inputs, and tail-based inputs
- **Requires**: Heavy Forwarder with Java runtime
- **When to use**: Direct database table/query ingestion; lookup table population from databases

### 2.7 Data Manager (Splunk Cloud Only)
- **Best for**: AWS, Azure, GCP cloud-native data sources on Splunk Cloud
- **How it works**: Splunk-managed service; auto-generates infrastructure templates (CloudFormation for AWS, ARM for Azure); configures best-practice data pipelines
- **Strengths**: Minutes instead of hours to configure, auto-generates cloud infrastructure, built-in health monitoring, centralized management UI
- **Limitations**: Splunk Cloud on AWS only (supported commercial regions), not all data sources supported
- **Recommended for**: High-volume cloud sources (>1TB/day) like Event Hubs, CloudWatch, GCS

### 2.8 Splunk OpenTelemetry Collector
- **Best for**: Kubernetes environments, cloud-native observability (metrics + traces + logs), multi-signal telemetry
- **How it works**: Distribution of the OpenTelemetry Collector with Splunk-specific receivers/exporters; deployed as DaemonSet in K8s or standalone agent; sends to Splunk via HEC
- **Strengths**: Vendor-agnostic, collects metrics/traces/logs in one agent, auto-discovery of apps/services, Helm chart deployment for K8s
- **When to use**: Kubernetes log/metric collection, cloud-native application observability, replacing multiple collection agents with one

### 2.9 Splunk Connect for SNMP (SC4SNMP)
- **Best for**: Network device performance monitoring via SNMP
- **How it works**: Containerized SNMP poller; deployed at network edge; polls devices for MIB data; sends to Splunk via HEC
- **Strengths**: Context-rich interface information, high availability design, no manual SNMP query construction
- **When to use**: Network infrastructure monitoring (CPU, memory, interface stats, device health)

---

## 3. Data Processing Pipeline (Pre-Ingest)

Modern Splunk deployments should evaluate these pre-ingest processing options:

### 3.1 Edge Processor
- **Deployment**: Customer-hosted (on-premises Linux node)
- **Function**: Filter, mask, transform, route data using SPL2 pipelines before it reaches Splunk
- **Data sources**: Receives from UFs (S2S) and syslog sources
- **Destinations**: Splunk indexes, Amazon S3, other systems
- **Included with**: Splunk Cloud Platform and Splunk Enterprise subscriptions (no additional cost)
- **Use cases**: PII masking, filtering verbose logs (e.g., Windows debug events), routing subsets to S3 for low-cost storage, reducing ingest volume

### 3.2 Ingest Processor
- **Deployment**: Splunk-hosted SaaS (Splunk Cloud only)
- **Function**: Same as Edge Processor plus log-to-metric conversion
- **Included with**: Splunk Cloud Platform (Essentials tier included)
- **Use cases**: Same as Edge Processor, plus converting high-cardinality logs to metrics for Splunk Observability Cloud

### 3.3 Ingest Actions
- **Deployment**: Built into Splunk Enterprise and Splunk Cloud
- **Function**: GUI-based rules for filtering, masking, routing at ingest time (uses props.conf/transforms.conf under the hood)
- **Use cases**: Quick data transformations without SPL2, PII removal, routing to S3

---

## 4. Major Splunk-Supported TAs by Domain

### 4.1 Operating Systems

| TA Name | Splunkbase ID | Collection Method | Key Sourcetypes | CIM Models | Notes |
|---------|--------------|-------------------|-----------------|------------|-------|
| Splunk Add-on for Windows | 742 | UF scripted inputs + WMI | `WinEventLog`, `XmlWinEventLog`, `WinRegistry`, `Perfmon`, `WinHostMon`, `WinNetMon`, `WinPrintMon` | Authentication, Change, Endpoint, Network Traffic, Performance | v5.0+ split WinEventLog into Classic/XML channels. Breaking changes — follow upgrade docs |
| Splunk Add-on for Unix and Linux | 833 | UF scripted inputs | `linux_syslog`, `linux_secure`, `linux_audit`, `linux_bootlog`, `cpu`, `df`, `interfaces`, `iostat`, `ps`, `top`, `vmstat`, `who` | Authentication, Change, Endpoint, Network Traffic, Performance | v6.0+ has breaking .conf changes — test before production upgrade |
| Splunk Add-on for Sysmon | - | UF (Windows Event Log input) | `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | Endpoint, Network Traffic, Change | Collects from Sysmon's dedicated Windows Event Log channel |
| TA for macOS | 8280 | UF file monitor | `macos:system:log`, `macos:install:log` | Authentication, Change, Endpoint | Monitors `/var/log/system.log` and `/var/log/install.log` |

### 4.2 Cloud Platforms

| TA Name | Splunkbase ID | Collection Method | Key Sourcetypes | Notes |
|---------|--------------|-------------------|-----------------|-------|
| Splunk Add-on for AWS | 1876 | Modular inputs (HF/IDM/SH) or Data Manager | `aws:cloudtrail`, `aws:cloudwatch`, `aws:config`, `aws:s3`, `aws:vpc:flow`, `aws:guardduty`, `aws:securityhub` | v7.0+ merges Amazon Security Lake capabilities. Supports SQS-based S3 input for high volume |
| Splunk Add-on for Microsoft Cloud Services | 3110 | Modular inputs (Event Hubs, Storage APIs) or Data Manager | `mscs:azure:eventhub`, `azure:monitor:aad`, `azure:monitor:activity`, `azure:monitor:resource`, `azure:storage:blob`, `azure:storage:table` | Primary TA for Azure infrastructure data. Uses Azure Service Management APIs and Azure Storage APIs |
| Splunk Add-on for Microsoft Office 365 | 4055 | Modular inputs (REST API) | `o365:management:activity`, `o365:service:status`, `o365:service:message` | Pulls from O365 Management Activity API and Service Communications API. Replaces the O365 modular input in MS Cloud Services TA |
| Splunk Add-on for Microsoft Azure | 6324 | Modular inputs or Event Hub push (via Azure Functions + HEC) | `azure:aad:signin`, `azure:aad:audit`, `azure:security:center:alert`, `azure:compute:vm` | Push approach (Azure Functions → HEC) recommended for real-time, high-volume scenarios |
| Splunk Add-on for Google Cloud Platform | 3088 | Modular inputs (Pub/Sub, REST API) or Data Manager | `google:gcp:pubsub:message`, `google:gcp:pubsub:audit:*`, `google:gcp:compute:*`, `google:gcp:billing:*` | v4.0+ introduced granular sourcetyping for Pub/Sub audit data |

**Azure Ingestion Decision Tree:**
1. **Splunk Cloud on AWS?** → Use **Data Manager** (auto-generates ARM templates, easiest setup)
2. **High volume (>1TB/day)?** → Use **Push via Azure Functions + HEC** or **Data Manager**
3. **Low-moderate volume, Splunk Enterprise?** → Use **MS Cloud Services TA** or **MS Azure TA** modular inputs on HF
4. **Need O365 audit/activity logs?** → Use **O365 TA** (dedicated, replaces the legacy input in MS Cloud Services TA)

### 4.3 Network & Security

| TA Name | Collection Method | Key Sourcetypes | Notes |
|---------|-------------------|-----------------|-------|
| Splunk Add-on for Cisco ASA | Syslog (SC4S or UF) | `cisco:asa` | Firewall logs via syslog. SC4S auto-identifies ASA |
| Cisco Networks Add-on | Syslog + SNMP | `cisco:ios`, `cisco:nx-os` | IOS/NX-OS device logs |
| Splunk Add-on for Palo Alto Networks | Syslog (SC4S or UF) + modular inputs | `pan:traffic`, `pan:threat`, `pan:system`, `pan:config`, `pan:globalprotect`, `pan:iot_alert`, `pan:iot_device`, `pan:xdr_incident` | Syslog input uses parent `pan:log` which auto-transforms to specific subtypes at index time. Also collects IoT Security and Cortex XDR via API |
| Splunk Add-on for CrowdStrike FDR | Modular inputs (S3/SQS) | `crowdstrike:events:sensor`, `crowdstrike:events:platform`, `crowdstrike:events:nge`, `crowdstrike:fdr:*` | Falcon Data Replicator pulls from CrowdStrike-managed S3 buckets |
| CrowdStrike Falcon Event Streams TA | Modular inputs (Streaming API) | `crowdstrike:event:streams:json` | Real-time streaming of detection and audit events |
| Okta Identity Cloud Add-on | Modular inputs (REST API) | `OktaIM2:log`, `OktaIM2:user`, `OktaIM2:group`, `OktaIM2:app` | Event logs, user/group/app data via Okta REST APIs |
| Zscaler Technical Add-On | Syslog (NSS) or Cloud NSS (HEC) | `zscalernss-web`, `zscalernss-fw`, `zscalernss-dns`, `zscalernss-tunnel` | CIM-mapped for ES. Cloud NSS sends directly via HEC |
| Splunk Add-on for Fortinet FortiGate | Syslog (SC4S or UF) | `fgt_log`, `fgt_traffic`, `fgt_utm`, `fgt_event` | Standard syslog collection from FortiGate appliances |
| Technology Add-on for NetFlow | HF with NFO (NetFlow Optimizer) | `netflow`, `sflow`, `ipfix` | Requires NetFlow Optimizer software. Also handles cloud flow logs (AWS VPC, Azure NSG, GCP VPC) |

### 4.4 Identity & Access Management

| TA Name | Collection Method | Key Sourcetypes | Notes |
|---------|-------------------|-----------------|-------|
| Splunk Add-on for Active Directory | UF (Windows) | `ActiveDirectory`, `MSAD:*` | Uses LDAP queries via scripted inputs on Windows UF |
| Okta Identity Cloud Add-on | Modular inputs (REST API) | `OktaIM2:log` | Event log, user, group, and app data |
| Splunk Add-on for Microsoft Entra ID | Via MS Azure TA or O365 TA | `azure:aad:signin`, `azure:aad:audit` | Sign-in and audit logs for Microsoft Entra ID (formerly Azure AD) |

### 4.5 Endpoint Detection & Response (EDR)

| TA Name | Collection Method | Key Sourcetypes | Notes |
|---------|-------------------|-----------------|-------|
| CrowdStrike FDR TA | S3/SQS modular input on HF | `crowdstrike:fdr:*` | High-volume endpoint telemetry |
| Splunk Add-on for Carbon Black | Modular inputs (REST API) | `carbonblack:*` | CB Response/Defense event data |
| Splunk Add-on for Symantec Endpoint Protection | Syslog or DB | `symantec:ep:*` | Syslog from SEPM or direct DB queries |
| Qualys TA | Modular inputs (REST API) | `qualys:hostDetection`, `qualys:was:scan`, `qualys:pc:*` | VM, WAS, PC, FIM, EDR, CSAM data |

### 4.6 IT Service Management

| TA Name | Collection Method | Key Sourcetypes | Notes |
|---------|-------------------|-----------------|-------|
| Splunk Add-on for ServiceNow | Modular inputs (REST API) | `snow:incident`, `snow:change_request`, `snow:cmdb_ci` | Bidirectional: also supports creating incidents/events from Splunk |
| Splunk Add-on for JIRA | Modular inputs (REST API) | `jira:issue`, `jira:changelog` | JIRA issue and changelog data |

### 4.7 Databases

| App Name | Collection Method | Key Sourcetypes | Notes |
|---------|-------------------|-----------------|-------|
| Splunk DB Connect | JDBC (scheduled query, rising column, tail) on HF | User-defined | Supports SQL Server, Oracle, MySQL, PostgreSQL, DB2, Sybase, and any JDBC-compatible database. Requires HF with Java |

### 4.8 Web & Application

| TA Name | Collection Method | Key Sourcetypes | Notes |
|---------|-------------------|-----------------|-------|
| Splunk Add-on for Apache Web Server | UF file monitor | `access_combined`, `apache:error` | Standard Apache access and error logs |
| Splunk Add-on for Nginx | UF file monitor | `nginx:plus:access`, `nginx:plus:error` | Nginx access and error logs |
| Splunk Add-on for IIS | UF file monitor | `ms:iis:auto` | IIS W3C extended log format |

### 4.9 Cisco Ecosystem (Relevant to Cisco-Splunk Integration)

| TA Name | Collection Method | Key Sourcetypes | Notes |
|---------|-------------------|-----------------|-------|
| Cisco Networks Add-on | Syslog | `cisco:ios`, `cisco:nx-os`, `cisco:wlc` | IOS, NX-OS, WLC device logs |
| Splunk Add-on for Cisco ASA | Syslog | `cisco:asa` | ASA/FTD firewall logs |
| Splunk Add-on for Cisco ESA | Syslog | `cisco:esa:*` | Email Security Appliance |
| Splunk Add-on for Cisco WSA | Syslog | `cisco:wsa:*` | Web Security Appliance |
| Splunk Add-on for Cisco ISE | Syslog + pxGrid | `cisco:ise:syslog` | ISE authentication/posture events |
| Cisco Meraki Add-on | Syslog + API | `meraki:*` | MX/MR/MS device events, client data |
| Splunk Add-on for Cisco Secure Firewall (FMC) | Syslog + eStreamer | `cisco:firepower:syslog` | Firepower/FMC events |
| Cisco Umbrella TA | Modular inputs (S3/API) | `cisco:umbrella:*` | DNS, proxy, cloud firewall logs |
| Cisco SD-WAN TA | Modular inputs (vManage API) | `cisco:sdwan:*` | SD-WAN telemetry and alarms |
| Cisco ThousandEyes TA | Modular inputs (REST API) | `thousandeyes:*` | Network and application test results |
| Cisco Webex TA | Modular inputs (REST API) | `cisco:webex:*` | Meeting, messaging, device telemetry |
| Cisco Spaces TA | Modular inputs (Firehose API) | `cisco:spaces:*` | Location analytics, occupancy, environmental |
| Splunk Add-on for Cisco Cyber Vision | Modular inputs or Syslog | `cisco:cybervision:*` | OT asset inventory, vulnerability, flow data |

---

## 5. Ingest Method Decision Framework

Use this decision tree when a new data source needs to be onboarded:

### Step 1: Identify the Data Source Type

```
Is the data source...
├── A file or directory on a host?
│   └── → Universal Forwarder (UF) with file/directory monitor
├── A Windows system?
│   └── → UF with Splunk Add-on for Windows
│       (event logs, perfmon, registry, AD)
├── A Linux/Unix system?
│   └── → UF with Splunk Add-on for Unix and Linux
│       (syslog, audit, performance scripts)
├── A syslog-emitting device (firewall, switch, appliance)?
│   └── → Splunk Connect for Syslog (SC4S)
│       Check SC4S supported sources list first
├── A cloud platform (AWS/Azure/GCP)?
│   └── → Check if Data Manager supports the source (Cloud only)
│       → Otherwise use platform-specific TA on HF/IDM/SH
├── A SaaS application with REST API?
│   └── → Vendor-specific TA with modular inputs on HF
│       (O365, Okta, ServiceNow, CrowdStrike, etc.)
├── A database (RDBMS)?
│   └── → Splunk DB Connect on HF
├── A Kubernetes cluster?
│   └── → Splunk OpenTelemetry Collector (Helm chart)
├── An application you control (custom code)?
│   └── → HTTP Event Collector (HEC) via SDK/REST
├── An SNMP-managed network device?
│   └── → Splunk Connect for SNMP (SC4SNMP)
├── A NetFlow/IPFIX/sFlow source?
│   └── → NetFlow TA + NetFlow Optimizer
└── An OT/IIoT protocol device (BACnet/Modbus/OPC-UA/MQTT)?
    └── → Cisco Edge Intelligence → Splunk OTI Cloud
        or custom HEC integration via protocol gateway
```

### Step 2: Check Splunkbase for Existing TA

1. Search `splunkbase.splunk.com` for the vendor/product name
2. **Prefer Splunk-supported TAs** (labeled "Splunk Supported") over community/developer-supported
3. Check the TA documentation for:
   - Supported sourcetypes and their CIM data model mappings
   - Required collection method (UF, HF, or HEC)
   - Splunk Cloud compatibility (IDM vs SH vs Victoria experience)
   - Version compatibility with your Splunk platform version
   - Known issues and breaking changes

### Step 3: Evaluate Pre-Ingest Processing Needs

Ask these questions:
- **Need to filter high-volume noise?** → Edge Processor or Ingest Actions
- **Need to mask PII before indexing?** → Edge Processor (SPL2 pipelines)
- **Need to route subsets to S3 for low-cost storage?** → Edge Processor
- **Need to convert logs to metrics?** → Ingest Processor (Splunk Cloud only)
- **Simple field transformations?** → Ingest Actions (GUI-based, simplest)

### Step 4: Validate CIM Compliance

For any data destined for Splunk Enterprise Security, ITSI, or CIM-dependent apps:
1. Verify the TA provides CIM-compatible field extractions, eventtypes, and tags
2. Install the Splunk CIM Add-on (`SA-CIM`) on search heads
3. Validate data appears in the appropriate CIM data model(s) using `| datamodel <model_name> search`
4. If the TA lacks CIM mappings, create custom `props.conf`/`transforms.conf` configurations or use the Add-on Builder

---

## 6. Splunk Cloud-Specific Considerations

| Component | Splunk Cloud Support | Notes |
|-----------|---------------------|-------|
| Universal Forwarder | Yes | Requires credentials package. Direct-to-Cloud or via intermediate forwarder |
| Heavy Forwarder | Yes (customer-managed) | Runs on-premises or in customer cloud account. Requires deployment server license from Splunk Support |
| HEC | Yes (native) | Available on Splunk Cloud indexers. Token management via Splunk Web |
| SC4S | Yes | Sends to Splunk Cloud via HEC |
| Data Manager (IDM) | Yes (Classic & Victoria) | Splunk-managed data collection node. Supports AWS/Azure/GCP sources. Does NOT support TCP/UDP/syslog inputs |
| Edge Processor | Yes | Customer-hosted node, Splunk-managed control plane. Included with subscription |
| Ingest Processor | Yes (SaaS) | Splunk-hosted. Essentials tier included with subscription |
| Modular Input TAs | Yes | Run on IDM (Classic) or SH/SHC (Victoria). Some TAs require filing a support ticket for IDM installation |
| Splunk OTel Collector | Yes | Sends to Splunk Cloud via HEC |

---

## 7. Best Practices for Data Pipeline Development

### 7.1 Data Onboarding Workflow (Splunk Lantern 5-Phase Model)
1. **Request**: Receive and document the data onboarding request (use case, source, volume estimate)
2. **Define**: Meet with requester to define the use case, validate event structure, timestamps, sourcetype elements; plan CIM normalization
3. **Implement**: Configure TA, inputs, indexes, field extractions, tags, reports, dashboards
4. **Validate**: Verify data accuracy (format, completeness, timestamp correctness), CIM compliance, field extraction quality
5. **Communicate**: Announce new data availability to the community (index, sourcetype, tags, knowledge objects)

### 7.2 Sourcetype Naming Conventions
- Use the TA's predefined sourcetypes — never override with custom names unless building a custom TA
- Format: `vendor:product:type` (e.g., `cisco:asa`, `pan:traffic`, `aws:cloudtrail`)
- Sourcetype drives all downstream parsing, CIM mapping, and dashboard functionality

### 7.3 Index Strategy
- Create dedicated indexes per data domain or retention requirement
- Never use the `main` index for production data
- Consider index naming conventions: `<env>_<domain>_<source>` (e.g., `prod_network_firewall`)

### 7.4 Timestamp Handling
- Verify the TA correctly extracts timestamps from events
- For HEC `/event` endpoint: ensure the `time` field is in the JSON payload, not buried in the event body
- For HEC `/raw` endpoint: configure `TIME_PREFIX` and `TIME_FORMAT` in `props.conf`
- If timestamps are unreliable, use `DATETIME_CONFIG = NONE` to use receipt time (last resort)

### 7.5 Avoiding Common Pitfalls
- **Duplicate field extractions**: Configure extractions at search time OR index time, never both (causes doubled values)
- **Sourcetype permissions**: Ensure custom sourcetype configurations are shared at the correct scope (app/global)
- **Modular input duplication**: Never deploy configured modular input TAs to multiple forwarders via deployment server (causes duplicate data)
- **HEC timestamp confusion**: The `/event` endpoint prioritizes the `time` field in the JSON envelope, not the event body timestamp

---

## 8. Quick Reference: Data Source → Recommended Ingest Path

| Data Source Category | Recommended Ingest | TA / App | Platform |
|---------------------|-------------------|----------|----------|
| Windows Event Logs | UF | Splunk Add-on for Windows | Enterprise + Cloud |
| Linux Syslog/Audit | UF | Splunk Add-on for Unix and Linux | Enterprise + Cloud |
| Firewalls (PAN, ASA, Fortinet) | SC4S (syslog) | Vendor-specific TA | Enterprise + Cloud |
| AWS CloudTrail/CloudWatch | Data Manager or TA on HF | Splunk Add-on for AWS | Cloud (DM) / Enterprise (HF) |
| Azure Activity/Monitor Logs | Data Manager or TA on HF | MS Cloud Services TA / Azure TA | Cloud (DM) / Enterprise (HF) |
| GCP Audit/Compute Logs | Data Manager or TA on HF | Splunk Add-on for GCP | Cloud (DM) / Enterprise (HF) |
| Office 365 Audit Logs | TA on HF/IDM/SH | Splunk Add-on for Microsoft Office 365 | Enterprise + Cloud |
| CrowdStrike FDR | TA on HF (S3/SQS) | CrowdStrike FDR TA | Enterprise + Cloud |
| Okta Events | TA on HF | Okta Identity Cloud Add-on | Enterprise + Cloud |
| ServiceNow | TA on HF | Splunk Add-on for ServiceNow | Enterprise + Cloud |
| RDBMS (SQL, Oracle, etc.) | DB Connect on HF | Splunk DB Connect | Enterprise + Cloud |
| Kubernetes Logs/Metrics | Splunk OTel Collector | OTel Helm chart | Enterprise + Cloud |
| Custom Application Logs | HEC (SDK/REST) | None needed (custom sourcetype) | Enterprise + Cloud |
| Network SNMP Metrics | SC4SNMP | SC4SNMP | Enterprise + Cloud |
| Network Flow (NetFlow/sFlow) | HF with NFO | NetFlow TA | Enterprise + Cloud |
| Cisco Network Devices | SC4S (syslog) | Cisco Networks Add-on | Enterprise + Cloud |
| Cisco Meraki | Syslog + API TA | Meraki Add-on | Enterprise + Cloud |
| Cisco ISE | Syslog + pxGrid | Splunk Add-on for Cisco ISE | Enterprise + Cloud |
| OT/IIoT (BACnet/Modbus) | Cisco Edge Intelligence → HEC | Splunk OTI Cloud / custom TA | Enterprise + Cloud |

---

## 9. Research Methodology

When asked about a data source not covered above:

1. **Search Splunkbase**: `https://splunkbase.splunk.com/apps?keyword=<vendor>` — check for Splunk-supported, developer-supported, and community TAs
2. **Check Splunk Lantern Data Sources**: `https://lantern.splunk.com/Data_Sources/<vendor>` — vendor-specific guidance with links to TAs, documentation, and use cases
3. **Check Splunk Docs**: `https://help.splunk.com/en/supported-add-ons/` — full documentation for Splunk-supported add-ons including sourcetypes, CIM mappings, and installation guides
4. **Check SC4S Source List**: `https://splunk.github.io/splunk-connect-for-syslog/` — verify if SC4S has an out-of-box parser for the syslog source
5. **Check Splunk Validated Architectures (SVA)**: `https://help.splunk.com/en/splunk-enterprise/splunk-validated-architectures/` — reference architectures for specific GDI scenarios
6. **If no TA exists**: Recommend building a custom TA using the **Splunk Add-on Builder** (Splunkbase ID 2962), ensuring CIM compliance and proper sourcetype naming

---

## 10. Lantern Data Sources Index

Splunk Lantern maintains vendor-specific data source pages with ingestion guidance. Key vendors covered:

Cisco, CrowdStrike, Dell, Docker, Fabrix.ai, Fortinet, Gigamon, GitHub, GitLab, Google, Kubernetes, Linux and Unix, Mac OS, Microsoft, NETSCOUT, Okta, OpenAI, Palo Alto Networks, Salesforce, SAP, Skyhigh Security, Symantec, Syslog, Tanium, Tenable, Trend Micro, VMware, Websense, Zeek, Zoom, Zscaler

Access pattern: `https://lantern.splunk.com/Data_Sources/<VendorName>`

---

*Last updated: March 2026. Always verify TA versions, compatibility, and breaking changes on Splunkbase before deployment.*
