---
name: splunk-lantern
description: >
  Splunk Lantern best-practices reference, distilled from lantern.splunk.com. Use this skill when:
  (1) Onboarding a new data source or sourcetype — props.conf Great 8, CIM compliance, data quality
  validation; (2) Writing or optimising SPL — filtering, tstats, join alternatives, eval chaining,
  summary indexing; (3) Implementing security use cases — Enterprise Security, RBA, MITRE ATT&CK,
  assets & identities, SOAR playbooks; (4) Building observability solutions — ITSI KPI design, RED/
  USE/LETS monitoring frameworks, SLO tracking; (5) Cloud data ingestion — AWS, Azure, Kubernetes,
  Edge Processor pipelines; (6) Dashboard best practices — base/chain searches, drilldown, tokens;
  (7) Platform governance — Splunk Success Framework, search head tuning, DMA optimisation.
---

# Splunk Lantern Best Practices Skill

Lantern (lantern.splunk.com) is Splunk's official practitioner knowledge base. This skill
translates its 200+ articles into copy-paste-ready patterns, SPL, and configuration snippets.

---

## 1. Data Onboarding — The "Great 8" props.conf Settings

Every new sourcetype needs all eight settings to eliminate parsing ambiguity and prevent
data quality issues at the source.

```ini
[my_sourcetype]
SHOULD_LINEMERGE        = false          # Never merge lines unless you explicitly need it
LINE_BREAKER            = ([\r\n]+)      # Where one event ends; use regex matching your data
TIME_PREFIX             = timestamp=     # Text immediately before the timestamp field
MAX_TIMESTAMP_LOOKAHEAD = 32             # Max chars to scan for timestamp after TIME_PREFIX
TIME_FORMAT             = %Y-%m-%dT%H:%M:%S  # strptime — %Y=4-digit year, %y=2-digit year
TRUNCATE                = 999999         # Prevent silent truncation of large events
EVENT_BREAKER_ENABLE    = true           # Enable intelligent event breaking (S2S/HEC)
EVENT_BREAKER           = ([\r\n]+)      # Mirror LINE_BREAKER for forwarder-side breaking
```

| Setting | Why it matters |
|---------|----------------|
| `SHOULD_LINEMERGE = false` | Merging consumes CPU; most structured logs don't need it |
| `TIME_PREFIX` | Without it, Splunk runs 30+ regexes against first 128 chars of every event |
| `TIME_FORMAT` | Prevents fallback to slow strptime auto-detection |
| `TRUNCATE = 999999` | Default 10000 silently cuts long JSON/XML events |

**Optional extras:**
```ini
CHARSET        = UTF-8   # Explicit beats auto-detect
ANNOTATE_PUNCT = false   # Saves processing; disable if not using punct field
```

### Configuring a New Sourcetype — Decision Checklist

1. Is there a Splunk-supported TA? → Use it, don't reinvent
2. Are events multi-line? → Set `SHOULD_LINEMERGE = true` + `BREAK_ONLY_BEFORE` regex
3. Non-standard timestamp? → Set all three: `TIME_PREFIX`, `TIME_FORMAT`, `MAX_TIMESTAMP_LOOKAHEAD`
4. Data via HEC/S2S? → Also set `EVENT_BREAKER_ENABLE` + `EVENT_BREAKER`
5. Events > 10 KB? → Set `TRUNCATE = 999999`

### HEC Timestamp Pitfall

If the `time` field is nested inside the event payload rather than at the HEC payload level,
Splunk defaults to ingestion time. Fix with `INGEST_EVAL`:

```ini
[my_hec_sourcetype]
TRANSFORMS-fixtime = extract_embedded_timestamp

[extract_embedded_timestamp]
INGEST_EVAL = _time=strptime(replace(_raw,".+\"time\":\s*\"(\d+)\".*","\\1"),"%s")
```

---

## 2. Data Quality Validation

**Detect parsing errors in _internal:**
```spl
index=_internal splunk_server=* source=*splunkd.log*
  (log_level=ERROR OR log_level=WARN)
  (component=AggregatorMiningProcessor OR component=DateParserVerbose OR component=LineBreakingProcessor)
| rex field=event_message "Context: source(::|=)(?<src>[^|]*?)\|host(::|=)(?<h>[^|]*?)\|(?<st>[^|]*?)\|"
| stats
    count(eval(component=="LineBreakingProcessor")) AS line_break_issues
    count(eval(component=="DateParserVerbose"))     AS timestamp_issues
    count(eval(component=="AggregatorMiningProcessor")) AS aggregation_issues
  BY st
| rename st AS sourcetype
| where line_break_issues>0 OR timestamp_issues>0 OR aggregation_issues>0
```

**Detect future-timestamped events (clock skew / wrong TIME_FORMAT):**
```spl
| tstats count AS events
    WHERE (_time>[| makeresults | eval now=now() | return $now]) index=*
  BY _time _indextime sourcetype span=10s
| eval delay_hrs=ceiling((_time-_indextime)/3600)
| stats max(delay_hrs) AS max_delay_hrs BY sourcetype
| where max_delay_hrs > 0
```

**Alert on missing sourcetypes — two-search pattern:**

*Weekly state-capture search (saves to lookup):*
```spl
| tstats count WHERE index=prod_* earliest=-60m latest=now BY sourcetype
| eval state="exists"
| append [| inputlookup sourcetype_state.csv]
| dedup sourcetype
| outputlookup sourcetype_state.csv
```

*Scheduled alert — fires when a sourcetype goes silent:*
```spl
| tstats count WHERE index=prod_* earliest=-60m latest=now BY sourcetype
| eval state="exists"
| append [| inputlookup sourcetype_state.csv | eval state="missing"]
| dedup sourcetype
| search state="missing"
```

---

## 3. CIM and Data Model Acceleration

### Why CIM matters

| Benefit | Detail |
|---------|--------|
| Faster searches | Queries TSIDX files, not raw events |
| ES detections | 90%+ of OOTB correlation searches require CIM data models |
| ITSI KPIs | `tstats` KPI base searches require accelerated models |
| Cross-source queries | One field name (`src`, `dest`, `user`) works across all sourcetypes |

**Key CIM data models and primary fields:**

| Data Model | Key fields | Common TA sources |
|------------|------------|-------------------|
| Network_Traffic | src, dest, src_port, dest_port, bytes_in, bytes_out | Palo Alto, Cisco ASA, Fortinet |
| Authentication | user, src, dest, action, app | Windows Security, Okta, LDAP |
| Web | src, dest, url, http_method, status, bytes | Apache, Nginx, Squid |
| Endpoint | user, dest, process, process_name, action | Sysmon, CrowdStrike, Carbon Black |
| Malware | dest, user, file_name, signature, action | AV/EDR products |
| IDS_Attacks | src, dest, signature, severity, ids_type | Snort, Suricata, Zeek |

**Validate CIM mapping:**
```spl
| tstats count FROM datamodel=Network_Traffic.All_Traffic BY All_Traffic.sourcetype
```

### DMA Optimisation — Priority Actions

1. **Disable unused models** — each enabled model runs scheduled searches every 5 min
2. **Constrain index scope** — edit `cim_<datamodel>_indexes` macros:
   ```ini
   [cim_network_traffic_indexes]
   definition = (index=network OR index=firewall)
   ```
3. **Shorten backfill** — set to 4–12 hours (default is often 3–7 days)
4. **Raise concurrency** in `limits.conf`:
   ```ini
   [search]
   max_searches_perc           = 75
   max_searches_perc_when_auth = 75
   ```
5. **Storage estimate:** `Daily ingest volume (GB) × 3.4 = additional storage needed per indexer`

**Check DMA coverage gaps:**
```spl
| tstats summariesonly=t count FROM datamodel=Network_Traffic BY index sourcetype
| append [| tstats count FROM datamodel=Network_Traffic BY index sourcetype]
| eval type=if(isnull(count),"not_accelerated","accelerated")
| stats values(type) AS types BY index sourcetype
```

---

## 4. SPL Optimisation — Patterns and Anti-Patterns

### Core rules (in priority order)

| Rule | Why |
|------|-----|
| Filter early: `index=`, `sourcetype=`, time range | Eliminates buckets before scanning |
| Use `fields` immediately after the first pipe | Reduces columns transferred to search head |
| Replace `join` with OR + conditional `stats` | `join` silently truncates at 50K events, 60s timeout |
| Replace `search` after `stats` with `where` | `where` runs locally; `search` re-distributes |
| Combine `eval` statements with commas | One eval call is cheaper than chaining multiple |
| Use `tstats` instead of `stats` where possible | Queries TSIDX, not raw events |
| Use `TERM()` for tokens with special characters | Prevents tokenisation splitting on `.` `=` `/` |

### join → OR + stats (most impactful anti-pattern fix)

```spl
-- ANTI-PATTERN: silently truncates at 50,000 events, 60s timeout
index=_internal sourcetype=splunkd component=Metrics
| stats count AS metrics BY host
| join host type=left
    [search index=_audit | stats count AS audits BY host]

-- BEST PRACTICE: single pass, no truncation
(index=_internal sourcetype=splunkd component=Metrics)
    OR (index=_audit sourcetype=audittrail)
| stats
    count(eval(sourcetype="splunkd"))    AS metrics
    count(eval(sourcetype="audittrail")) AS audits
  BY host
```

### Use fields early

```spl
index=myindex sourcetype=access_combined
| fields host, clientip, status, bytes, uri_path
| stats avg(bytes) AS avg_bytes count BY status, host
```

### Combine eval statements

```spl
-- ANTI-PATTERN
| eval category=if(status<400,"ok","error")
| eval mb=round(bytes/1024/1024,2)

-- BEST PRACTICE
| eval category=if(status<400,"ok","error"), mb=round(bytes/1024/1024,2)
```

### tstats for indexed-field queries

```spl
-- Replaces: index=_internal | stats count BY splunk_server
| tstats count WHERE index=_internal BY splunk_server

-- With PREFIX for non-default indexed fields (Splunk 8.0+)
| tstats count WHERE index=network PREFIX(dest_port=) BY host
```

### TERM() for literal token matching

```spl
-- Without TERM: "average=0.9" may split on "=" and "." 
index=metrics average=0.9*

-- With TERM: forces literal match, faster
index=metrics TERM(average=0.9*)
```

### Summary indexing pattern

```spl
-- Scheduled hourly search writes pre-aggregated data
index=web sourcetype=access_combined earliest=-1h@h latest=@h
| stats count AS requests sum(bytes) AS total_bytes avg(bytes) AS avg_bytes BY host, status
| collect index=summary_web sourcetype=web_hourly_summary

-- Dashboard queries tiny summary index
index=summary_web sourcetype=web_hourly_summary earliest=-7d
| timechart span=1h sum(requests) AS requests BY host
```

---

## 5. Search Head and Platform Performance

**Detect indexer data imbalance:**
```spl
| tstats count WHERE index=_internal BY splunk_server
```
All indexers should have roughly equal counts. A 1% imbalance can cause a 15% runtime
increase during peak concurrency.

**Find expensive scheduled searches:**
```spl
index=_audit action=search info=completed
| stats avg(total_run_time) AS avg_s, max(total_run_time) AS max_s, count AS runs
    BY savedsearch_name, user
| where avg_s > 60
| sort -avg_s
```

**Audit globally-scoped automatic lookups (should be app-scoped):**
```spl
| rest /servicesNS/-/-/data/props/lookups count=0 splunk_server=local
| where sharing="global"
| table title, eai:acl.app, stanza
```

**Audit unused calculated fields:**
```spl
| rest /servicesNS/-/-/data/props/calcfields count=0 timeout=900 splunk_server=local
    search=eai:acl.removable=1 f=stanza f=eai:*
| rename title AS Name, eai:acl.app AS App, stanza AS sourcetype
| table Name, App, sourcetype
```

### limits.conf tuning reference

```ini
# limits.conf on search heads
[search]
max_searches_perc           = 75   # Default 50; raise carefully
max_searches_perc_when_auth = 75
max_rt_search_multiplier    = 1    # Limit real-time searches

[subsearch]
maxout   = 100000   # Default 10000 — raise if subsearches truncate
maxtime  = 120      # Default 60s

[lookups]
max_matches = 1000
```

### Large lookup optimisation

| Problem | Solution |
|---------|----------|
| CSV > 50K rows slows bundle replication | Move to KV Store with `replicate = false` |
| Lookup applied to all events | Scope `LOOKUP-` stanza to specific sourcetype only |
| Repeated inputlookup calls | Cache once with `inputlookup` + `append`, or use KV Store |

**Disable KV Store on indexers:**
```ini
# server.conf on indexer nodes
[kvstore]
disabled = true
```

---

## 6. Enterprise Security — RBA, Detection, and Data

### Risk-Based Alerting (RBA)

RBA accumulates risk scores on objects (users, hosts, IPs) before creating notable events —
reduces alert volume 50–90% with higher fidelity.

**Audit RBA adoption (% of detections contributing risk):**
```spl
| rest /services/saved/searches
| search action.correlationsearch.enabled=1 NOT disabled=1
| eval has_risk=if(action.risk=1,"yes","no")
| stats count BY has_risk
```

**Inspect risk scores for a user over 7 days:**
```spl
index=risk object_type=user object="jdoe@corp.com" earliest=-7d
| stats sum(risk_score) AS total_risk, values(source) AS detections, dc(source) AS det_count
    BY object
| sort -total_risk
```

**ES 8.x terminology changes:**

| ES 7.x | ES 8.x |
|--------|--------|
| Notable Event | Finding |
| Risk Event | Intermediate Finding |
| Correlation Search | Detection |
| Incident Review | Finding Viewer |

### Assets and Identities — critical fields

**Assets (devices):**

| Field | Priority |
|-------|----------|
| `ip` | Required — primary match key |
| `nt_host` / `dns` | Hostname resolution |
| `priority` | critical/high/medium/low — drives urgency |
| `category` | pci, dmz, server, workstation |
| `owner` | Person or team responsible |

**Identities (users):**

| Field | Priority |
|-------|----------|
| `identity` | Required — username or email |
| `first` / `last` | Display name |
| `email` | Secondary match key |
| `priority` | critical/high/medium/low |
| `category` | admin, service_account, contractor |

**Validate asset lookup is populated:**
```spl
| inputlookup asset_lookup | stats count BY priority | sort priority
```

### MITRE ATT&CK coverage audit

```spl
| rest /services/saved/searches
| search action.correlationsearch.enabled=1 NOT disabled=1
| mvexpand annotations
| rex field=annotations "\"mitre_attack\":\[(?P<techniques>[^\]]+)\]"
| eval techniques=split(replace(techniques,"\"",""),",")
| mvexpand techniques
| stats dc(title) AS detection_count BY techniques
| sort -detection_count
```

### Security detection SPL patterns

```spl
-- Hosts with active malware (NIST SI-3)
| tstats count FROM datamodel=Malware.Malware_Attacks
    WHERE Malware_Attacks.action=allowed BY Malware_Attacks.dest
| rename Malware_Attacks.dest AS dest
| lookup asset_lookup ip AS dest OUTPUT priority, owner

-- IDS attack categories (NIST SI-4)
| tstats count FROM datamodel=Intrusion_Detection.IDS_Attacks
    WHERE IDS_Attacks.ids_type=network
  BY IDS_Attacks.category, IDS_Attacks.severity
| rename IDS_Attacks.* AS *
| sort -count
```

---

## 7. SOAR — Playbook Design (I2A2 Framework)

| Element | Description | Example |
|---------|-------------|---------|
| **Inputs** | What you start with | Phishing email, suspicious IP |
| **Interactions** | Services and people involved | VirusTotal, AD, analyst |
| **Actions** | What the playbook does | IP lookup, block, disable account |
| **Artifacts/Outcomes** | What you learn | IP is malicious → block confirmed |

**SOAR Adoption Maturity Model:**

| Stage | Description |
|-------|-------------|
| 1 | Basic single-tool automation, no branching |
| 2 | Multi-tool playbooks with decision logic |
| 3 | Cross-team orchestration, SLA tracking |
| 4 | AI/ML-driven triage, self-tuning (top 5% of SOCs) |

**Top 5 OOTB SOAR playbooks (Lantern-recommended):**
1. Recorded Future Indicator Enrichment
2. Phishing Investigate and Respond (90 min → 60 sec)
3. CrowdStrike Malware Triage
4. Recorded Future Correlation Response
5. MS Graph / Office 365 Search and Purge

**Workbook vs Playbook:**
- **Workbook** = complete use case (e.g., "handle phishing end-to-end")
- **Playbook** = single automated step (e.g., "enrich the sender IP")

---

## 8. ITSI — KPI Design and Observability Frameworks

### KPI design rules (Lantern "Definitive Guide")

| Rule | Rationale |
|------|-----------|
| Limit to 3–6 KPIs per service | >10 KPIs dilutes health score sensitivity |
| Use KPI base searches over ad hoc | One shared search per metric family reduces scheduler load |
| Apply SRE Golden Signals (LETS) | Latency, Errors, Traffic, Saturation — covers services and infra |
| Enable predictive analytics only with >5 KPIs + 1 week history | ML needs enough signal |
| Don't enable all NEAPs simultaneously | Creates alert storms; tune one policy at a time |

### Three monitoring frameworks

**RED** (request-scoped — microservices):

| Signal | What to measure |
|--------|-----------------|
| Rate | Requests/sec — watch spikes AND drops |
| Errors | Client 4xx and server 5xx separately |
| Duration | p50, p95, p99 — not just average |

**USE** (resource-scoped — infrastructure):

| Signal | What to measure |
|--------|-----------------|
| Utilisation | % time resource is busy |
| Saturation | Queue depth, wait time |
| Errors | Hardware errors, dropped packets |

**LETS / Golden Signals** (applies to both):

| Signal | Example KPI |
|--------|------------|
| Latency | `avg(response_time_ms)` by service |
| Errors | `error_rate = errors/total * 100` |
| Traffic | requests per second |
| Saturation | CPU%, memory%, queue depth |

### ITSI introspection SPL

```spl
-- Services with no entity members (N/A health score — entity rules broken)
index=itsi_summary kpi_id="SHKPI-*" entity_key="N/A" earliest=-15m
| stats count BY service_title
| sort -count

-- KPI search failures
index=_internal sourcetype=scheduler
  savedsearch_name="Indicator*" OR savedsearch_name="Shared*"
  status!=success
| stats count BY savedsearch_name, reason
| sort -count
```

### Custom ITSI threshold template (conf seed)

```ini
[cpu_workhours_template]
title = CPU Threshold - Work Hours Only
time_variate_thresholds          = True
adaptive_thresholds_is_enabled   = False
```

---

## 9. Cloud Data Ingestion — Method Selection

### AWS ingestion hierarchy

| Method | Type | Use when |
|--------|------|----------|
| **Data Manager** | Push via Firehose | Preferred — automated, low maintenance |
| Splunk Add-on for AWS | Pull (CloudWatch/S3) | Data Manager unavailable |
| Amazon Data Firehose → HEC | Push streaming | High-volume; no pre-ingest transformation |

> Always confirm **egress charges** with cloud admin teams before enabling.

### Azure ingestion hierarchy

| Method | Type | Use when |
|--------|------|----------|
| **Event Hubs + Data Manager** | Push | Preferred |
| Event Hubs + Azure Functions | Push | Custom transformation needed |
| Splunk Add-on for Microsoft Cloud Services | Pull | Legacy / simple environments |

**Event Hub input tuning — one stanza per partition:**
```ini
[azure_event_hubs://partition_0]
consumer_group = splunk
partition_id   = 0
thread_count   = 1
```

### Re-ingesting S3 data written by ingest actions

```ini
# props.conf
[aws:s3:reingested]
SHOULD_LINEMERGE = false
TRUNCATE         = 2000000
TIME_PREFIX      = "time":
TIME_FORMAT      = %s
TRANSFORMS-meta  = s3_sourcetype_override, s3_source_override

# transforms.conf
[s3_sourcetype_override]
DEST_KEY = MetaData:Sourcetype
REGEX    = "sourcetype":"(?<sourcetype>[^"]*)"
FORMAT   = sourcetype::$1
```

### Kubernetes / OpenTelemetry via Helm

```yaml
splunkPlatform:
  token:    <HEC_TOKEN>
  endpoint: https://<splunk_host>:8088/services/collector
  index:    k8s_logs
splunkObservability:
  accessToken: <O11Y_TOKEN>
  realm:       us1
clusterName: production-k8s
logsCollection:
  enabled: true
  containers:
    excludeAgentLogs: true
```

---

## 10. Edge Processor — SPL2 Pipeline Patterns

Edge Processor filters, masks, enriches, and routes data before it leaves your network.
Uses SPL2 syntax (not SPL).

### Cisco ASA noise reduction

```spl2
import 'cisco_msg_id.csv' from /envs.splunk.'<env_id>'.lookups
import route from /splunk/ingest/commands

| from $source
| rex field=_raw /(?P<_raw>(%ASA|%FTD).*)/
| rex field=_raw /(%ASA|%FTD)-\d+-(?P<msg_id>\d+)/
| where msg_id != "302013"   /* TCP connection log - very noisy */
| where msg_id != "302015"   /* UDP connection log - very noisy */
| lookup 'cisco_msg_id.csv' msg_id AS msg_id OUTPUT explanation
| route
    (where msg_id like "1%"  -> index: firewall_critical, sourcetype: cisco:asa)
    (where true()            -> index: firewall_general,  sourcetype: cisco:asa)
```

### PII masking

```spl2
| from $source
| rex mode=sed field=_raw "s/\b\d{3}-\d{2}-\d{4}\b/REDACTED_SSN/g"
| rex mode=sed field=_raw "s/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b/REDACTED_EMAIL/g"
```

### Licence consumption measurement via INGEST_EVAL

```ini
# props.conf
[default]
TRANSFORMS-measure = capture_event_size

# transforms.conf
[capture_event_size]
INGEST_EVAL = event_length=len(_raw)
```

```spl
| tstats sum(event_length) AS total_bytes
    WHERE index=* host=ep-node-*
    _index_earliest=-30d@d _index_latest=-1d@d
  BY host _time span=1d@d
| eval GB=round(total_bytes/1024/1024/1024,3)
| timechart span=1d sum(GB) AS daily_GB BY host
```

**Scale trigger:** Add EP nodes when CPU or memory sustains > 70% utilisation.

---

## 11. Dashboard Best Practices

### Base and chain searches (Classic XML)

```xml
<search id="base_traffic">
  <query>
    index=network sourcetype=firewall
    | stats sum(bytes_in) AS bytes_in sum(bytes_out) AS bytes_out count AS sessions
        BY src_ip, dest_ip, dest_port
  </query>
  <earliest>$time.earliest$</earliest>
  <latest>$time.latest$</latest>
</search>

<!-- Panel chaining off the base search -->
<chart>
  <search base="base_traffic">
    <query>| stats sum(bytes_in) AS total_in BY src_ip | sort -total_in | head 10</query>
  </search>
</chart>
```

> Base search limits: 50,000 events max; 30-second limit for non-transforming queries.

### Drilldown with loadjob (avoid re-running expensive searches)

```xml
<!-- Capture SID on completion -->
<chart>
  <search id="source_search">
    <query>index=large_dataset | stats ...</query>
  </search>
  <drilldown>
    <set token="detail_sid">$job.sid$</set>
    <link target="_blank">/app/myapp/detail?sid=$detail_sid$</link>
  </drilldown>
</chart>

<!-- Detail dashboard reuses the cached job -->
<search>
  <query>| loadjob $sid$ | search $click.value$</query>
</search>
```

### Dashboard Studio — chain search pattern

```json
{
  "dataSources": {
    "ds_base": {
      "type": "ds.search",
      "options": {
        "query": "index=network | stats count sum(bytes) AS bytes BY src_ip dest_ip"
      }
    },
    "ds_top_talkers": {
      "type": "ds.chain",
      "options": {
        "extend": "ds_base",
        "query": "| stats sum(bytes) AS total BY src_ip | sort -total | head 10"
      }
    }
  }
}
```

---

## 12. Splunk Success Framework Reference

### Four functional areas

| Area | Key activities |
|------|----------------|
| **Program Management** | RACI, executive sponsor, QBRs, value realization |
| **People Management** | RBAC, welcome pages, role-based training |
| **Platform Management** | Architecture, capacity planning, backup/restore |
| **Data Management** | Lifecycle policy, onboarding governance, CIM compliance |

### Four outcome paths

| Path | Focus |
|------|-------|
| **Reduce Costs** | DMA optimisation, summary indexing, index tiering, search efficiency |
| **Improve Performance** | Search head tuning, limits.conf, lookup cleanup |
| **Increase Efficiencies** | Data onboarding standards, Cloud migration readiness |
| **Mitigate Risk** | Backup/restore, audit logging, alert coverage gap analysis |

### Adoption maturity levels

| Level | Characteristics |
|-------|----------------|
| Foundational | Basic data in, ad hoc searches, manual dashboards |
| Standard | CIM compliance, scheduled alerts, basic ES/ITSI |
| Intermediate | RBA, SOAR integration, data model acceleration |
| Advanced | ML/AI detections, full-stack observability, automated remediation |

### Cloud migration approach selection

| Approach | Downtime | Best for |
|----------|----------|---------|
| **Dual-fire** | None | Large, risk-averse environments |
| **Greenfield** | Config window only | Fresh start, config migration only |
| **Cutover** | Required | Small, simple environments |

**Pre-migration checklist:**
- [ ] UFs at supported version (9.x recommended)
- [ ] SAML/LDAP auth tested against Cloud IdP
- [ ] Custom roles audited — Cloud restricts some capabilities
- [ ] Apps AppInspected for Cloud compatibility
- [ ] Index naming inventory complete
- [ ] Saved searches using `rest` command reviewed

---

## 13. Quick Reference SPL Patterns (Lantern Distilled)

| Task | SPL |
|------|-----|
| Event indexing delay check | `\| tstats max(_indextime) AS it WHERE index=* BY sourcetype \| eval delay_s=it-_time \| stats max(delay_s) BY sourcetype` |
| Indexer load balance | `\| tstats count WHERE index=_internal BY splunk_server` |
| Sourcetype volume trend | `\| tstats count WHERE index=* BY sourcetype _time span=1d \| timechart span=1d sum(count) BY sourcetype` |
| Expensive searches by user | `index=_audit action=search info=completed \| stats avg(total_run_time) AS avg_s count BY user savedsearch_name \| where avg_s>60 \| sort -avg_s` |
| DMA health check | `\| tstats summariesonly=t count FROM datamodel=Network_Traffic BY sourcetype` |
| ITSI KPI failures | `index=_internal sourcetype=scheduler (savedsearch_name="Indicator*" OR savedsearch_name="Shared*") status!=success \| stats count BY savedsearch_name, reason` |
| RBA top risk objects | `index=risk earliest=-24h \| stats sum(risk_score) AS score BY object, object_type \| sort -score \| head 20` |
| Lookup file audit | `\| rest /servicesNS/-/-/data/transforms/lookups count=0 \| table title, filename, eai:acl.app` |
| Benford's Law (fraud) | `index=transactions amount>0 \| eval d1=substr(tostring(floor(amount)),1,1) \| stats count BY d1 \| eventstats sum(count) AS total \| eval expected_pct=round(log(1+1/tonumber(d1))/log(10)*100,2), actual_pct=round(count/total*100,2) \| eval deviation=abs(actual_pct-expected_pct) \| where deviation>5` |
