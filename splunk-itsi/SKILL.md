---
name: splunk-itsi
description: >
  Splunk ITSI developer and admin reference. Use this skill when: (1) Building or configuring
  ITSI services, KPIs, entities, glass tables, or teams via the UI or REST API; (2) Writing
  KPI base searches or threshold configurations; (3) Automating ITSI object management with
  REST API or Python — services, entities, episodes, aggregation policies; (4) Troubleshooting
  ITSI issues — N/A health scores, skipped KPI searches, pseudo-entities, episode grouping;
  (5) Designing service dependency trees and health score weighting; (6) Configuring itsi_settings.conf,
  itsi_kpi_base_search.conf, or threshold templates; (7) Packaging ITSI content for deployment
  via content packs or backup/restore; (8) Working with itsi_summary, itsi_tracked_alerts, or
  itsi_grouped_alerts indexes.
---

# Splunk ITSI Developer and Admin Reference

ITSI stores nearly all configuration in the **KV Store** (not `.conf` files), calculates
weighted health scores from KPI severity values, and correlates notable events into grouped
episodes. Never write directly to KV Store collections — always use the REST API at
`/servicesNS/nobody/SA-ITOA/itoa_interface/`.

---

## 1. Architecture and Data Storage

### Data flow: raw events → actionable episodes

```
Raw data → Splunk indexes
  → Scheduled KPI searches (every N min) → itsi_summary
  → Threshold evaluation → severity (Normal/Low/Medium/High/Critical)
  → Health score = weighted avg of KPI severities → SHKPI-<service_id> in itsi_summary
  → Correlation searches → notable events → itsi_tracked_alerts
  → Rules Engine (NATS) → episodes grouped → itsi_grouped_alerts
  → Episode Review (triage)
```

### ITSI-specific indexes

| Index | Content | Default retention |
|-------|---------|-------------------|
| `itsi_summary` | KPI results: severity, alert_value, entity data | Configurable |
| `itsi_tracked_alerts` | Notable events with full metadata | 6 months / 500K events |
| `itsi_grouped_alerts` | Episode grouping data (group_id, event_id) | Configurable |
| `itsi_notable_archive` | Archived notable events from KV Store | Long-term |

### KV Store vs .conf files

| Stored in KV Store (via REST API) | Stored in .conf files only |
|-----------------------------------|---------------------------|
| Services, KPIs, entities, glass tables | Severity definitions |
| Deep dives, teams, templates | Retention policies |
| Correlation searches, aggregation policies | Global settings |
| Episodes, event metadata | Notable event status labels |
| KPI templates, threshold templates (seed → KV) | Threshold label score contributions |

### Installed app components

| App | Purpose |
|-----|---------|
| `SA-ITOA` | Core engine, REST APIs, Rules Engine |
| `itsi` | Frontend UI |
| `DA-ITSI-*` | Domain adapters / content packs |
| `SA-IndexCreation` | Creates ITSI indexes |
| `SA-UserAccess` | Role management |

**Default RBAC roles:** `itoa_admin`, `itoa_analyst`, `itoa_user`

---

## 2. Services

### Service JSON structure

```json
{
  "_key": "auto-generated-uuid",
  "object_type": "service",
  "title": "Email Service",
  "description": "Corporate email infrastructure",
  "enabled": 1,
  "sec_grp": "default_itsi_security_group",
  "entity_rules": [
    {
      "rule_items": [
        {
          "field": "host",
          "field_type": "alias",
          "rule_type": "matches",
          "value": "mail-server-*"
        }
      ],
      "rule_condition": "AND"
    }
  ],
  "kpis": [],
  "services_depends_on": [
    {
      "service_id": "dependent-service-uuid",
      "kpis_depending_on": ["kpi-uuid"]
    }
  ],
  "service_tags": {
    "tags": ["production", "tier1"],
    "template_tags": []
  },
  "base_service_template_id": "<template_key>"
}
```

### Entity rule logic

- Top-level array elements joined by **OR** — any matching rule group adds the entity
- Within each rule group, `rule_items` joined by `rule_condition` (AND or OR)
- `field_type`: `alias` (matches KPI search results) or `info` (metadata field)
- `rule_type`: `matches` or `not`
- `value`: supports `*` wildcards; empty value matches everything

### Health score formula

```
Health Score = Σ(severity_score × importance) / Σ(importance)
```

**Severity scores:** Normal=**100**, Low=**70**, Medium=**50**, High=**30**, Critical=**0**
(Info severity is excluded from health score calculation)

**Importance values:**

| Value | Meaning |
|-------|---------|
| `0` | Excluded from health score entirely |
| `1–10` | Weighted normally in formula |
| `11` | **Minimum health indicator** — if this KPI is Critical, the entire service becomes Critical regardless of other KPIs |

**Example calculation:**
```
KPI-A: severity=Normal(100), importance=10
KPI-B: severity=Medium(50),  importance=7
KPI-C: severity=Low(70),     importance=5
Health = (100×10 + 50×7 + 70×5) / (10+7+5) = 1700/22 = 77.3
```

### Service dependencies

- `services_depends_on` defines upstream dependencies
- Dependent service health score KPI defaults to **importance 11**
- Critical upstream service forces parent to critical
- Cyclic dependencies display as dotted lines in Service Analyzer — avoid them
- `| getservice` SPL command returns services with direct dependencies

### Service CRUD via REST API

**Base URL:** `https://<host>:8089/servicesNS/nobody/SA-ITOA/itoa_interface/service`

**Create:**
```bash
curl -k -u admin:changeme \
  https://localhost:8089/servicesNS/nobody/SA-ITOA/itoa_interface/service \
  -H "Content-Type: application/json" -X POST \
  -d '{
    "object_type": "service",
    "title": "Web Application Service",
    "enabled": 1,
    "sec_grp": "default_itsi_security_group",
    "entity_rules": [
      {"rule_items": [{"field": "host", "field_type": "alias",
        "rule_type": "matches", "value": "webapp-*"}], "rule_condition": "AND"}
    ],
    "kpis": [], "services_depends_on": []
  }'
```

**List (with field filtering):**
```bash
curl -k -u admin:changeme \
  "https://localhost:8089/servicesNS/nobody/SA-ITOA/itoa_interface/service?fields=title,_key&limit=50"
```

**Partial update** — always use `is_partial_data=1` or you overwrite the entire object:
```bash
curl -k -u admin:changeme \
  "https://localhost:8089/servicesNS/nobody/SA-ITOA/itoa_interface/service/<_key>/?is_partial_data=1" \
  -X POST -H "Content-Type: application/json" \
  -d '{"description": "Updated description"}'
```

**Bulk disable:**
```bash
curl -k -u admin:changeme \
  "https://localhost:8089/servicesNS/nobody/SA-ITOA/itoa_interface/service/bulk_update?is_partial_data=1" \
  -H "Content-Type: application/json" -X POST \
  -d '[{"_key": "service-key-1", "enabled": 0}, {"_key": "service-key-2", "enabled": 0}]'
```

**Delete by key:**
```bash
curl -k -u admin:changeme \
  https://localhost:8089/servicesNS/nobody/SA-ITOA/itoa_interface/service/<_key> -X DELETE
```

---

## 3. KPIs

### Four KPI search types

| Type | `search_type` value | Use when |
|------|---------------------|---------|
| Ad-hoc | `adhoc` | Simple, single-service; avoid at scale |
| Shared base search | `shared_base` | **Production best practice** — one search powers multiple KPIs |
| Data model | `datamodel` | Smaller environments with accelerated data models |
| Metrics | `metrics` | High-volume numeric data in metrics indexes (`mstats`) |

**Shared base search performance:** 4 KPIs sharing one base search = **129 executions/day** instead of **516**

### KPI scheduling parameters

| Parameter | Description | Typical value |
|-----------|-------------|---------------|
| `alert_period` | Run frequency (minutes) | `5` |
| `search_alert_earliest` | Lookback window (minutes) | `5–15` |
| `alert_lag` | Indexing delay offset (seconds) | `30` |

Effective search window: `earliest=-<search_alert_earliest>m+<alert_lag>s latest=-<alert_lag>s`

Named in Splunk as: `Indicator - <KPI_name> - <ID>` (ad-hoc) or `Shared - <n> - <ID>` (base)

### KPI importance and alert policy

| `alert_on` value | Behaviour |
|-----------------|-----------|
| `aggregate` | Notable event at service level only |
| `entity` | Notable event per entity only |
| `both` | Notable events at both levels |

### Threshold types

**Static:** Manual numeric ranges per severity level. Set direction: increase-only, decrease-only, or both.

**Time-variate:** Different threshold ranges per time block. Only one policy active at a time.
Templates: work hours, off-hours, weekends, AM/PM, 2-hour or 3-hour blocks.

**Adaptive (ML-based):** Algorithms auto-calculate thresholds from historical data.
- Requires **≥ 7 days backfill data**
- Recalculates nightly at midnight
- Algorithms: Standard Deviation (`stl`), Quantile (`quantile`), Range (`range`)
- ITSI 4.17+: ML-Assisted Thresholding recommends algorithm and values automatically
- Anomaly detection deprecated in ITSI 4.20 — replaced by adaptive thresholding with outlier detection

### KPI threshold templates

- Reusable configurations applied to multiple KPIs
- Updates to a template **automatically propagate** to all linked KPIs
- Exist only in the Global team
- Managed via Configuration → KPI Threshold Templates or REST `/itoa_interface/kpi_threshold_template`

### Example KPI SPL searches

**CPU utilization:**
```spl
index=os sourcetype=cpu host=*
| stats avg(cpu_load_percent) as cpu_load_percent by host
```

**Memory usage:**
```spl
index=os sourcetype=vmstat
| stats avg(memUsedPct) as mem_used_percent by host
```

**Disk free space:**
```spl
index=os sourcetype=df
| stats min(FreePct) as min_free_pct by host, filesystem
```

**Network throughput and errors:**
```spl
index=os sourcetype=interfaces
| stats avg(speed) as throughput_kbps, sum(iferrors) as total_errors by host, Name
```

**HTTP availability:**
```spl
index=web_monitoring sourcetype=web_ping
| stats count as total, count(eval(status>=200 AND status<400)) as success by host, url
| eval availability_pct=round(success/total*100,2)
```

**Shared base search powering three KPIs (one execution):**
```spl
index=os sourcetype=cpu OR sourcetype=vmstat OR sourcetype=df
| stats avg(cpu_load_percent) as cpu_pct,
        avg(memUsedPct) as mem_pct,
        min(FreePct) as disk_free_pct
  by host
```

### Create a KPI base search via REST

```bash
curl -k -u admin:changeme \
  https://localhost:8089/servicesNS/nobody/SA-ITOA/itoa_interface/kpi_base_search \
  -H "Content-Type: application/json" -X POST \
  -d '{
    "title": "OS Metrics Base Search",
    "base_search": "index=os sourcetype=cpu OR sourcetype=vmstat | stats avg(cpu_load_percent) as cpu_pct, avg(memUsedPct) as mem_pct by host",
    "alert_period": "5",
    "search_alert_earliest": "5",
    "alert_lag": "30",
    "is_entity_breakdown": true,
    "entity_breakdown_id_fields": "host",
    "is_service_entity_filter": true,
    "entity_alias_filtering_fields": "host",
    "metrics": [
      {"title": "cpu_pct", "threshold_field": "cpu_pct", "unit": "%",
       "aggregate_statop": "avg", "entity_statop": "avg"},
      {"title": "mem_pct", "threshold_field": "mem_pct", "unit": "%",
       "aggregate_statop": "avg", "entity_statop": "avg"}
    ],
    "sec_grp": "default_itsi_security_group"
  }'
```

### Monitor KPI search performance

```spl
index=_internal sourcetype=scheduler savedsearch_name="Indicator*" OR savedsearch_name="Shared*"
| stats count as run_count,
        count(eval(status!="success")) as failed_count,
        avg(run_time) as avg_runtime,
        max(run_time) as max_runtime
  by savedsearch_name
| eval kpi_type=if(savedsearch_name like "%Shared%","base","adhoc")
| sort -max_runtime
```

---

## 4. Entities

### Entity JSON structure

```json
{
  "_key": "8b12efff-d81d-409e-8607-35d504e7b4a1",
  "title": "web-server-01",
  "object_type": "entity",
  "identifier": {
    "fields": ["host", "ip"],
    "values": ["web-server-01", "192.168.1.10"]
  },
  "informational": {
    "fields": ["os", "datacenter", "owner"],
    "values": ["Linux", "US-East", "TeamA"]
  },
  "entity_type_ids": ["nix"]
}
```

**Field types:**
- `identifier` (aliases) — used to match entities to KPI search results
- `informational` — descriptive metadata, not used for matching

**Entity statuses:** Active, Inactive, Unstable, N/A

### Five entity import methods

| Method | When to use |
|--------|------------|
| Manual (UI) | One-off additions |
| CSV import | Bulk import up to 50,000 entities; conflict modes: Skip, Update/Merge, Replace |
| Search-based import (one-time) | SPL discovery; saved search title must match `"IT Service Intelligence - <text>"` |
| Recurring import (scheduled) | Creates `ITSI Import Objects - <importName>` cron searches |
| Content pack integrations | OOTB discovery for Unix, Windows, VMware, AWS EC2, Kubernetes |

### Entity aliasing and pseudo-entities

- **Entity Filter Field**: field in KPI search results used to match entities
- **Entity Split Field**: field used for per-entity breakdown
- When Filter ≠ Split fields, an **entity pivot** occurs (filter by datacenter, split by host)
- If split field values don't match real entity aliases → **pseudo-entities** created (`entity_key=N/A`)
  - Appear in Service Analyzer but are not clickable
  - Fix by aligning split field values to actual entity alias values

### Entity REST API

**Create:**
```bash
curl -k -u admin:changeme \
  https://localhost:8089/servicesNS/nobody/SA-ITOA/itoa_interface/entity \
  -H "Content-Type: application/json" -X POST \
  -d '{
    "title": "web-server-01",
    "object_type": "entity",
    "identifier": {"fields": ["host", "ip"], "values": ["web-server-01", "192.168.1.10"]},
    "informational": {"fields": ["os", "datacenter"], "values": ["Linux", "US-East"]}
  }'
```

**Filter by regex:**
```bash
curl -k -u admin:changeme \
  'https://localhost:8089/servicesNS/nobody/SA-ITOA/itoa_interface/entity/?fields=title,_key&filter={"title":{"$regex":".*mysql"}}'
```

**Bulk update (payload must be a JSON array):**
```bash
curl -k -u admin:changeme \
  "https://localhost:8089/servicesNS/nobody/SA-ITOA/itoa_interface/entity/bulk_update?is_partial_data=1" \
  -H "Content-Type: application/json" -X POST \
  -d '[{"_key": "entity-guid-1", "cpu": ["56"]}, {"_key": "entity-guid-2", "cpu": ["72"]}]'
```

**List all alias field names:**
```bash
curl -k -u admin:changeme \
  https://localhost:8089/servicesNS/nobody/SA-ITOA/itoa_interface/get_alias_list
```

**Query entities via SPL:**
```spl
| rest splunk_server=local /servicesNS/nobody/SA-ITOA/itoa_interface/entity
    fields="_key,title,identifier,informational" report_as=text
| eval value=spath(value,"{}")
| mvexpand value
| eval entity_title=spath(value,"title"),
       entity_aliases=mvzip(spath(value,"identifier.fields{}"),spath(value,"identifier.values{}"),"=")
| table entity_title, entity_aliases
```

---

## 5. Configuration Files

**Rule:** Never modify `default/` directories — always override in `local/`.

### itsi_kpi_base_search.conf

```ini
[<stanza_name>]
title                          = <string>
base_search                    = <SPL>
alert_period                   = <integer>      # Run frequency in minutes
search_alert_earliest          = <integer>      # Lookback window in minutes
alert_lag                      = <integer>      # Indexing delay offset (default 30)
is_entity_breakdown            = <bool>
entity_breakdown_id_fields     = <string>       # e.g., "host"
entity_alias_filtering_fields  = <csv-list>
is_service_entity_filter       = <bool>
```

### itsi_kpi_template.conf

```ini
[<stanza_name>]
title          = <string>
description    = <string>
_owner         = itsi
kpis           = <json>         # JSON array of KPI definitions
source_itsi_da = <string>       # Source module / content pack
```

### itsi_kpi_threshold_template.conf

```ini
[<stanza_name>]
title                                  = <string>
time_variate_thresholds                = [True|False]
adaptive_thresholds_is_enabled         = [True|False]
adaptive_thresholding_training_window  = -7d    # Minimum 7 days
time_variate_thresholds_specification  = <JSON>
```

### itsi_settings.conf

```ini
[default]
disabled = 1    # "1" ENABLES staggered KPI search scheduling — KEEP THIS SET

[backup_restore]
job_queue_timeout = 43200   # 12 hours default

[import]
import_batch_size = 1000
```

### Other key conf files

| File | Purpose |
|------|---------|
| `itsi_notable_event_retention.conf` | KV Store archival (default: 6 months / 500K events). Events in open episodes never archived |
| `itsi_notable_event_severity.conf` | Episode severity levels 1–6 with color mappings |
| `threshold_labels.conf` | Score contribution values per severity level |

---

## 6. REST API — Complete Reference

### Authentication methods

```bash
# Basic auth
curl -k -u admin:password ...

# Session key
SESSION=$(curl -k -u admin:password https://localhost:8089/services/auth/login \
  -d username=admin -d password=password | grep -o '<sessionKey>[^<]*' | cut -d'>' -f2)
curl -k -H "Authorization: Splunk $SESSION" ...

# Bearer token (recommended for automation)
curl -k -H "Authorization: Bearer <token>" ...
```

All POST requests require `-H "Content-Type: application/json"`.
SPL `| rest` calls to SA-ITOA require `report_as=text` (ITSI 4.4.0+).

### ITOA Interface endpoint map

Base: `/servicesNS/nobody/SA-ITOA/itoa_interface/`

| Endpoint | GET | POST | DELETE | Notes |
|----------|-----|------|--------|-------|
| `/<object_type>` | List all | Create | Filter delete | Collection CRUD |
| `/<object_type>/<_key>` | Get one | Update one | Delete one | Single object |
| `/<object_type>/bulk_update` | — | Bulk create/update | — | Payload must be JSON array |
| `/<object_type>/count` | Count | — | — | |
| `/get_supported_object_types` | List all types | — | — | |
| `/get_alias_list` | All entity alias fields | — | — | |
| `/service/<_key>/templatize` | Generate template | — | — | |
| `/entity/restore` | — | Restore retired | — | |

**Supported object types:** `team`, `entity`, `service`, `base_service_template`,
`kpi_base_search`, `deep_dive`, `glass_table`, `home_view`, `kpi_template`,
`kpi_threshold_template`, `event_management_state`, `entity_type`,
`entity_filter_rule`, `custom_threshold_windows`

### Event Management Interface endpoint map

Base: `/servicesNS/nobody/SA-ITOA/event_management_interface/`

| Endpoint | Description |
|----------|-------------|
| `/notable_event_group` | Episodes CRUD |
| `/notable_event_aggregation_policy` | Aggregation policies |
| `/correlation_search` | Correlation searches |
| `/notable_event_comment` | Episode comments |
| `/notable_event_email_template` | Email templates |
| `/ticketing` | External ticket creation |

**Other interfaces:**
- `maintenance_services_interface/maintenance_calendar` — maintenance windows
- `backup_restore_interface/backup_restore` — backup/restore jobs

### GET query parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `fields` | String | Comma-separated fields to return |
| `filter` | String | MongoDB-style JSON filter (URL-encoded) |
| `limit` | Integer | Max entries to return |
| `offset` | Integer | Entries to skip (pagination) |
| `sort_key` | String | Field to sort by |
| `sort_dir` | Integer | `1`=ascending, `0`=descending |
| `is_partial_data` | Integer | `1` = partial update (preserve existing fields) |

### MongoDB-style filter syntax

```json
{"title": "exact match"}
{"title": {"$regex": ".*web.*"}}
{"enabled": 1}
{"$and": [{"enabled": 1}, {"title": {"$regex": ".*prod.*"}}]}
{"severity": {"$gte": "5"}}
```

### Episode management

**Status codes:** 1=New, 2=In Progress, 3=Pending, 4=Resolved, 5=Closed
**Severity codes:** 1=Info, 2=Normal, 3=Low, 4=Medium, 5=High, 6=Critical

**Update episode status:**
```bash
curl -k -u admin:changeme \
  "https://localhost:8089/servicesNS/nobody/SA-ITOA/event_management_interface/notable_event_group/<episode_id>/?is_partial_data=1" \
  -X POST -H "Content-Type: application/json" \
  -d '{"status": "2", "owner": "admin"}'
```

**Update episode severity:**
```bash
curl -k -u admin:changeme \
  "https://localhost:8089/servicesNS/nobody/SA-ITOA/event_management_interface/notable_event_group/<episode_id>/?is_partial_data=1" \
  -X POST -H "Content-Type: application/json" \
  -d '{"severity": "6"}'
```

### Critical API gotchas

| Gotcha | Detail |
|--------|--------|
| `is_partial_data=1` required for updates | Without it, POST completely overwrites the object |
| Bulk payloads must be JSON arrays | Even for single object: `[{"_key": "...", "field": "value"}]` |
| DELETE with incorrect filter deletes ALL | Always prefer deleting by `_key` |
| Nested array replacement | Partial update replaces entire `kpis` array — read, modify, write back |
| `report_as=text` required | Mandatory for SPL `\| rest` calls to SA-ITOA (v4.4.0+) |

### Python SDK pattern

```python
import requests, json, urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

BASE = "https://localhost:8089/servicesNS/nobody/SA-ITOA/itoa_interface"
AUTH = ("admin", "password")

# List services
services = requests.get(f"{BASE}/service",
    auth=AUTH, verify=False,
    params={"fields": "title,_key,enabled", "limit": 50}).json()

# Create entity
entity = {
    "title": "db-server-01",
    "object_type": "entity",
    "identifier": {"fields": ["host"], "values": ["db-server-01"]},
    "informational": {"fields": ["role", "env"], "values": ["database", "production"]}
}
result = requests.post(f"{BASE}/entity",
    auth=AUTH, verify=False,
    headers={"Content-Type": "application/json"},
    data=json.dumps(entity)).json()

# Bulk update (payload MUST be a JSON array)
updates = [{"_key": "guid-1", "cpu_cores": ["8"]},
           {"_key": "guid-2", "cpu_cores": ["16"]}]
requests.post(f"{BASE}/entity/bulk_update",
    auth=AUTH, verify=False,
    headers={"Content-Type": "application/json"},
    params={"is_partial_data": 1},
    data=json.dumps(updates))

# Get episodes (high severity)
EM = "https://localhost:8089/servicesNS/nobody/SA-ITOA/event_management_interface"
episodes = requests.get(f"{EM}/notable_event_group",
    auth=AUTH, verify=False,
    params={"limit": 20, "filter": json.dumps({"severity": {"$gte": "5"}})}).json()
```

---

## 7. Glass Tables and Notable Events

### Glass table data source JSON

```json
{
  "dataSources": {
    "ds_qoxtTg6k": {
      "type": "ds.search",
      "name": "test_service - CPU KPI",
      "options": {
        "query": "`get_full_itsi_summary_kpi(544bb528161b0decad02ad69)` `service_level_kpi_only` | timechart cont=false latest(alert_value) AS alert_value"
      },
      "meta": {
        "kpiID": "544bb528161b0decad02ad69",
        "serviceID": "d1b1e040-fa2e-4f28-bc7b-b618f3b91afe"
      }
    }
  }
}
```

### Notable Event Aggregation Policies (NEAPs)

- Group events into episodes based on: filtering criteria, split-by fields, breaking criteria, automated action rules
- **Split-by fields:** common values — `host`, `serviceid`, `itsi_service_id`
- **Breaking criteria:** e.g., no events for 8 hours
- **Default Aggregation Policy:** catches all unmatched events — cannot be deleted or disabled
- **Smart Mode:** ML-based grouping to automatically identify similar events

### Episode lifecycle

```
New → In Progress → Pending → Resolved → Closed
```

### Severity levels (notable events and episodes)

| Code | Label | Color |
|------|-------|-------|
| 1 | Info | Grey |
| 2 | Normal | Green |
| 3 | Low | Yellow-green |
| 4 | Medium | Yellow |
| 5 | High | Orange |
| 6 | Critical | Red |

### Pre-built correlation searches (Monitoring & Alerting Content Pack)

- Sustained Service Health Degradation
- Sustained KPI Degradation
- Sustained Entity KPI Degradation
- Multi-Service Episode Storm
- Episode Risk Anomaly

---

## 8. Troubleshooting Reference

### Service health stuck at N/A

**Cause:** No entities matched by service entity rules, or entity alias mismatch.

**Diagnose:**
```spl
index=itsi_summary kpi_id="SHKPI-*" entity_key="N/A" earliest=-15m
| stats count BY service_title
| sort -count
```

**Check entity membership:**
```spl
index=itsi_summary kpi_id="SHKPI-*" earliest=-15m
| stats dc(entity_key) AS entity_count, values(entity_key) AS entities BY service_title
| where entity_count=0 OR (entity_count=1 AND entities="N/A")
```

**Fix:** Verify entity alias values match exactly what your KPI search returns in the split field.
Avoid wildcards in entity aliases — they create pseudo-entities.

### KPI not calculating / skipped searches

**Diagnose skipped/failed KPI searches:**
```spl
index=_internal sourcetype=scheduler
  (savedsearch_name="Indicator*" OR savedsearch_name="Shared*")
  (status=skipped OR status=failed OR status=error)
| stats count AS occurrences, values(reason) AS reasons BY savedsearch_name, status
| sort -occurrences
```

**Find KPI searches running over their period:**
```spl
index=_internal sourcetype=scheduler savedsearch_name="Indicator*" OR savedsearch_name="Shared*"
| stats avg(run_time) as avg_rt, max(run_time) as max_rt BY savedsearch_name
| where max_rt > 300
| sort -max_rt
```

**Common causes:** Search takes longer than `alert_period`, search head concurrency limit hit,
base search overloaded, missing `report_as=text` on `| rest`.

### Pseudo-entities

**Cause:** KPI split-by field values don't match real entity alias values.

**Diagnose:**
```spl
index=itsi_summary entity_key="N/A" earliest=-1h
| stats count BY itsi_service_id, kpiid
| sort -count
```

**Fix:** Align entity alias values to the exact string your KPI search produces in the split field.

### itsi_summary gaps

**Diagnose missing KPI data:**
```spl
index=itsi_summary earliest=-1h
| bucket _time span=5m
| stats dc(kpiid) AS kpi_count BY _time, itsi_service_id
| where kpi_count < 1
```

**Check KV Store collection sizes:**
```spl
| rest /services/admin/kvstore-collection-stats
| table name, count, size
| sort -size
```

### Event indexing delay

```spl
| tstats max(_indextime) AS indexed_time count
    WHERE index=itsi_summary latest=now earliest=-1h
  BY itsi_service_id
| eval delay_s=indexed_time-_time
| stats max(delay_s) AS max_delay_s BY itsi_service_id
| sort -max_delay_s
```

---

## 9. Best Practices

### Service design

- Start with **1–2 high-value services** tied to critical business transactions
- Limit KPIs to **3–6 per service** (maximum 20) for meaningful health score sensitivity
- Use **Service Sandbox** (ITSI 4.19+) to test without impacting production
- Break services into sub-services when entities behave differently (by data center, batch vs. general-purpose)
- Validate dependency hierarchy in Service Analyzer tree view before go-live

### KPI and search optimization

- **Always prefer shared base searches** over ad-hoc at scale (4 KPIs → 129 vs 516 executions/day)
- Monitor the **50,000 row limit** in `limits.conf` (`max_action_results`) — increase proportionally
- Row count formula: `services × KPIs × entities + services × 2`
- Keep `itsi_settings.conf` staggering enabled — disabling causes all KPI searches to fire simultaneously
- Use ITSI Health Check dashboard to monitor base search execution times

### Threshold tuning workflow

1. Define organizational severity meanings before setting thresholds
2. Start with **only Normal and Critical** using static thresholds
3. Enable adaptive thresholding temporarily to learn historical values
4. Switch back to static using learned ranges
5. Validate in Deep Dives against known past incidents
6. If a KPI is Critical >75% of the time, the threshold is wrong
7. Set non-critical KPIs to **Info** severity — visible in Deep Dives but excluded from health score

### Scaling considerations

- Batch REST API at **500–1,000 objects per call** for large entity environments
- Avoid wildcards in entity aliases — causes entity broadcasting and performance issues
- Reduce entity filter string length for large-member services
- Tune `itsi_notable_event_retention.conf` (default 6 months)
- Monitor KV Store collection sizes via ITSI Health Check dashboard

### Naming conventions

| Object | Convention | Example |
|--------|-----------|---------|
| Services | Business-meaningful, environment-scoped | `Online Payment Gateway - US East` |
| KPIs | Descriptive with unit | `CPU Load Percent`, `Transaction Error Rate` |
| Entity identifiers | Match exactly what your data produces | `web-server-01` not `Web Server 01` |
| Content pack objects | Prefix with pack identifier | `CP-OS-CPU Load` |

### Deployment packaging

| Method | Best for |
|--------|---------|
| ITSI Backup/Restore | KV Store migration across environments; enable "Include .conf files" toggle |
| Authored Content Packs (4.20+) | Bundle ITSI + Splunk knowledge objects into installable `.spl` |
| REST API export/import | Programmatic migration — GET from source, POST to target |

---

## 10. Quick-Reference SPL Patterns

| Task | SPL |
|------|-----|
| Services with N/A health | `index=itsi_summary kpi_id="SHKPI-*" entity_key="N/A" earliest=-15m \| stats count BY service_title` |
| Skipped KPI searches | `index=_internal sourcetype=scheduler (savedsearch_name="Indicator*" OR savedsearch_name="Shared*") status=skipped \| stats count BY savedsearch_name` |
| KPI severity distribution | `index=itsi_summary kpi_id!="SHKPI-*" earliest=-1h \| stats count BY alert_level \| sort -count` |
| KPI search runtime audit | `index=_internal sourcetype=scheduler savedsearch_name="Indicator*" OR savedsearch_name="Shared*" \| stats avg(run_time) AS avg_s, max(run_time) AS max_s BY savedsearch_name \| where avg_s>60 \| sort -avg_s` |
| Episode review summary | `index=itsi_grouped_alerts earliest=-24h \| stats count BY status, severity \| sort severity` |
| KV Store collection sizes | `\| rest /services/admin/kvstore-collection-stats \| table name, count, size \| sort -size` |
| All services list | `\| rest splunk_server=local /servicesNS/nobody/SA-ITOA/itoa_interface/service fields=title,_key,enabled report_as=text count=0 \| spath input=value path="{}" \| table title, _key, enabled` |
| Entity alias field inventory | `curl -k -u admin:pass https://localhost:8089/servicesNS/nobody/SA-ITOA/itoa_interface/get_alias_list` |
