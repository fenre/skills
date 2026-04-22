---
name: splunk-admin
description: "Comprehensive Splunk administration guide covering data ingestion, knowledge objects, SPL searching, and dashboards. Use when: (1) Configuring data inputs and parsing (inputs.conf, props.conf, transforms.conf), (2) Creating or managing knowledge objects (saved searches, macros, lookups, field extractions, data models), (3) Writing SPL queries for analysis, alerts, or reports, (4) Building dashboards in Simple XML or Dashboard Studio, (5) Troubleshooting Splunk configuration or search performance"
---

# Splunk Administration Guide

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA SOURCES                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  Files/Logs    │  Network      │  APIs         │  Agents                    │
│  • monitor://  │  • TCP/UDP    │  • HEC        │  • Universal Forwarder     │
│  • batch://    │  • Syslog     │  • REST       │  • Heavy Forwarder         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DATA PIPELINE                                        │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐          │
│  │  Input    │ →  │  Parsing  │ →  │  Indexing │ →  │  Search   │          │
│  │ (inputs)  │    │ (props/   │    │ (indexes) │    │  (SPL)    │          │
│  │           │    │ transforms)│    │           │    │           │          │
│  └───────────┘    └───────────┘    └───────────┘    └───────────┘          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      KNOWLEDGE OBJECTS                                       │
│  Saved Searches │ Field Extractions │ Lookups │ Macros │ Data Models        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      VISUALIZATION                                           │
│  Dashboards (Simple XML / Dashboard Studio) │ Reports │ Alerts              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Ingestion

### Configuration File Locations

| File | Purpose | Location Priority |
|------|---------|-------------------|
| `inputs.conf` | Data input definitions | Forwarders, Indexers |
| `props.conf` | Parsing rules, timestamps, field extractions | Indexers (index-time), Search Heads (search-time) |
| `transforms.conf` | Complex transforms, routing, lookups | Indexers, Search Heads |
| `outputs.conf` | Data forwarding destinations | Forwarders |
| `indexes.conf` | Index definitions | Indexers |

**Configuration Precedence:** `$SPLUNK_HOME/etc/system/local/` > `$SPLUNK_HOME/etc/apps/<app>/local/` > `$SPLUNK_HOME/etc/apps/<app>/default/` > `$SPLUNK_HOME/etc/system/default/`

### inputs.conf

Defines data sources to monitor and ingest.

**File Monitoring:**
```ini
[monitor:///var/log/myapp.log]
disabled = false
index = main
sourcetype = myapp_logs
host_segment = 3

[monitor:///var/log/nginx/*.log]
whitelist = \.log$
blacklist = \.gz$
followTail = 0
crcSalt = <SOURCE>
```

**Network Inputs:**
```ini
[tcp://1514]
sourcetype = syslog
index = network
connection_host = dns

[udp://514]
sourcetype = syslog
index = network
no_appending_timestamp = true
```

**HTTP Event Collector:**
```ini
[http]
disabled = 0
enableSSL = 1
port = 8088

[http://mytoken]
disabled = 0
index = main
indexes = main,summary
sourcetype = httpevent
```

**Scripted Inputs:**
```ini
[script:///opt/splunk/bin/scripts/myscript.sh]
interval = 300
sourcetype = custom_script
index = scripted
disabled = false
```

**Batch (One-time Ingest):**
```ini
[batch:///data/archive/*.log]
move_policy = sinkhole
sourcetype = archive_logs
```

### props.conf

Controls parsing, timestamps, and search-time extractions.

**Timestamp Extraction:**
```ini
[mysourcetype]
TIME_FORMAT = %Y-%m-%d %H:%M:%S.%3N
TIME_PREFIX = timestamp=
MAX_TIMESTAMP_LOOKAHEAD = 30
TZ = UTC
```

**Line Breaking:**
```ini
[mysourcetype]
LINE_BREAKER = ([\r\n]+)
SHOULD_LINEMERGE = false
TRUNCATE = 10000
```

**Field Extraction (Search-Time):**
```ini
[mysourcetype]
# Inline extraction
EXTRACT-user = user=(?<username>\w+)

# Using transforms.conf
REPORT-fields = my_field_extraction

# Key-Value extraction
KV_MODE = auto
```

**JSON Handling:**
```ini
[json_sourcetype]
KV_MODE = json
INDEXED_EXTRACTIONS = json
TIMESTAMP_FIELDS = timestamp,@timestamp
```

**Calculated Fields:**
```ini
[mysourcetype]
EVAL-duration_seconds = duration_ms / 1000
EVAL-status_category = case(status<400,"success", status<500,"client_error", true(),"server_error")
```

**Field Aliases:**
```ini
[mysourcetype]
FIELDALIAS-user = username AS user
FIELDALIAS-src = source_ip AS src
```

### transforms.conf

Complex transformations, routing, and lookups.

**Field Extraction:**
```ini
[my_field_extraction]
REGEX = ^(?<timestamp>\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2})\s+\[(?<level>\w+)\]\s+(?<message>.*)$
FORMAT = timestamp::$1 level::$2 message::$3
```

**Index Routing:**
```ini
[route_by_severity]
REGEX = level=(ERROR|CRITICAL)
DEST_KEY = _MetaData:Index
FORMAT = errors

# In props.conf:
# TRANSFORMS-routing = route_by_severity
```

**Host Override:**
```ini
[extract_host]
REGEX = host=(\S+)
DEST_KEY = MetaData:Host
FORMAT = host::$1
```

**Data Masking:**
```ini
[mask_credit_card]
REGEX = (\d{4})-(\d{4})-(\d{4})-(\d{4})
FORMAT = XXXX-XXXX-XXXX-$4
DEST_KEY = _raw
```

**Event Routing to Null Queue:**
```ini
[drop_debug_events]
REGEX = level=DEBUG
DEST_KEY = queue
FORMAT = nullQueue

# In props.conf:
# TRANSFORMS-filter = drop_debug_events
```

**Lookup Definition:**
```ini
[user_lookup]
filename = users.csv
match_type = WILDCARD(username)
max_matches = 1
min_matches = 1
default_match = unknown
```

### Applying Configuration Changes

```bash
# Validate configuration
$SPLUNK_HOME/bin/splunk btool props list --debug
$SPLUNK_HOME/bin/splunk btool transforms list --debug

# Reload without restart (search-time changes)
curl -k -u admin:password https://localhost:8089/services/data/props/_reload
curl -k -u admin:password https://localhost:8089/services/data/transforms/_reload

# Full restart required for index-time changes
$SPLUNK_HOME/bin/splunk restart
```

---

## Knowledge Objects

### Saved Searches

Create in `savedsearches.conf` or via UI.

```ini
[My Alert - High Error Rate]
search = index=main level=ERROR | stats count | where count > 100
cron_schedule = */5 * * * *
dispatch.earliest_time = -5m
dispatch.latest_time = now
enableSched = 1
alert_type = number of events
alert_comparator = greater than
alert_threshold = 0
alert.suppress = 1
alert.suppress.period = 1h
actions = email
action.email.to = admin@example.com
```

### Macros

Reusable search snippets in `macros.conf`.

```ini
# Simple macro
[my_index]
definition = index=main OR index=summary

# Macro with arguments
[time_filter(2)]
args = field, minutes
definition = $field$ > relative_time(now(), "-$minutes$m")

# Usage: `my_index` `time_filter(_time, 30)`
```

### Lookups

**CSV Lookup:**
```ini
# transforms.conf
[asset_lookup]
filename = assets.csv
max_matches = 1

# props.conf
[mysourcetype]
LOOKUP-assets = asset_lookup ip AS src_ip OUTPUT asset_name, location
```

**Automatic Lookup:**
```ini
# transforms.conf
[geo_lookup]
filename = geo.csv
match_type = CIDR(ip_range)

# props.conf  
[network_logs]
LOOKUP-geo = geo_lookup ip AS client_ip OUTPUT country, city
```

### Field Extractions

**Inline (props.conf):**
```ini
[sourcetype]
EXTRACT-fields = (?<action>\w+)\s+by\s+(?<user>\S+)
```

**Transform-based:**
```ini
# transforms.conf
[extract_kv]
REGEX = (\w+)=([^\s,]+)
FORMAT = $1::$2
MV_ADD = true

# props.conf
[sourcetype]
REPORT-kv = extract_kv
```

### Data Models

Hierarchical schemas for tstats acceleration.

```json
{
  "modelName": "MyDataModel",
  "objects": [
    {
      "objectName": "BaseEvent",
      "constraints": [{"search": "index=main"}],
      "fields": [
        {"fieldName": "user", "type": "string"},
        {"fieldName": "action", "type": "string"},
        {"fieldName": "duration", "type": "number"}
      ]
    }
  ]
}
```

**Accelerated Search:**
```spl
| tstats count FROM datamodel=MyDataModel WHERE nodename=BaseEvent BY user, action
```

---

## SPL Search Reference

### Search Pipeline Structure

```
<base_search> | <command1> | <command2> | ... | <output_command>
```

### Essential Commands

| Command | Purpose | Example |
|---------|---------|---------|
| `search` | Filter events | `search status=404` |
| `where` | Filter with expressions | `where count > 100` |
| `eval` | Create/modify fields | `eval gb = bytes/1024/1024/1024` |
| `stats` | Aggregate statistics | `stats count avg(duration) by host` |
| `timechart` | Time-series aggregation | `timechart span=1h count by status` |
| `chart` | Pivot table aggregation | `chart count over host by status` |
| `table` | Display specific fields | `table _time, host, message` |
| `rex` | Extract fields with regex | `rex field=_raw "user=(?<user>\w+)"` |
| `spath` | Extract from JSON/XML | `spath path=data.value output=value` |
| `rename` | Rename fields | `rename src_ip AS source` |
| `fields` | Include/exclude fields | `fields + host, source, - _raw` |
| `dedup` | Remove duplicates | `dedup host, source` |
| `sort` | Sort results | `sort - count` (descending) |
| `head/tail` | First/last N results | `head 100` |
| `top/rare` | Most/least common values | `top 10 user` |
| `transaction` | Group related events | `transaction session_id maxspan=30m` |
| `join` | Join datasets | `join type=left host [search index=assets]` |
| `lookup` | Enrich with lookup | `lookup users.csv user OUTPUT department` |
| `append` | Add results | `append [search index=other]` |
| `appendpipe` | Fork and append | `appendpipe [stats count]` |
| `eventstats` | Stats without aggregation | `eventstats avg(duration) as avg_dur` |
| `streamstats` | Running calculations | `streamstats sum(count) as running_total` |

### Statistical Functions

| Function | Description |
|----------|-------------|
| `count(field)` | Count non-null values |
| `dc(field)` | Distinct count |
| `sum(field)` | Sum of values |
| `avg(field)` | Average |
| `min(field)` / `max(field)` | Minimum/Maximum |
| `median(field)` | Median value |
| `perc95(field)` | 95th percentile |
| `stdev(field)` | Standard deviation |
| `values(field)` | All distinct values (MV) |
| `list(field)` | All values including duplicates |
| `first(field)` / `last(field)` | Chronological first/last |
| `earliest(field)` / `latest(field)` | Earliest/latest by _time |

### Eval Functions

**String Functions:**
```spl
| eval lower_user = lower(user)
| eval domain = split(email, "@")
| eval short = substr(message, 1, 50)
| eval clean = trim(field)
| eval replaced = replace(text, "old", "new")
| eval len = len(message)
```

**Numeric Functions:**
```spl
| eval rounded = round(value, 2)
| eval absolute = abs(diff)
| eval ceiling = ceiling(value)
| eval log_val = log(value, 10)
| eval power = pow(base, exp)
```

**Time Functions:**
```spl
| eval epoch = strptime(time_str, "%Y-%m-%d %H:%M:%S")
| eval formatted = strftime(_time, "%Y-%m-%d")
| eval hour = strftime(_time, "%H")
| eval relative = relative_time(now(), "-1d@d")
| eval diff = _time - start_time
```

**Conditional Functions:**
```spl
| eval status = if(code < 400, "success", "error")
| eval category = case(
    code < 200, "info",
    code < 300, "success", 
    code < 400, "redirect",
    code < 500, "client_error",
    true(), "server_error"
  )
| eval value = coalesce(field1, field2, "default")
| eval result = nullif(value, "N/A")
```

**Multivalue Functions:**
```spl
| eval combined = mvappend(field1, field2)
| eval count = mvcount(mv_field)
| eval first = mvindex(mv_field, 0)
| eval filtered = mvfilter(match(mv_field, "pattern"))
| eval sorted = mvsort(mv_field)
| eval deduped = mvdedup(mv_field)
```

### Time Modifiers

| Modifier | Description |
|----------|-------------|
| `-1h` | Relative: 1 hour ago |
| `-7d@d` | 7 days ago, snap to day start |
| `@w0` | Snap to Sunday |
| `earliest=-24h latest=now` | Last 24 hours |
| `earliest=01/01/2024:00:00:00` | Absolute time |

### Search Optimization

**Use indexed fields first:**
```spl
# Good - uses indexed fields
index=main sourcetype=access_combined status=500

# Avoid - field not indexed
index=main | search response_code=500
```

**Limit time range:**
```spl
index=main earliest=-1h latest=now
```

**Use tstats for accelerated data models:**
```spl
| tstats count FROM datamodel=Web WHERE Web.status>=500 BY Web.src
```

**Avoid subsearches with large results:**
```spl
# Better: Use join or lookup instead of large subsearches
| join type=left host [| inputlookup assets.csv]
```

---

## Dashboards

### Simple XML Structure

```xml
<dashboard version="1.1">
  <label>My Dashboard</label>
  <description>Dashboard description</description>
  
  <search id="base_search">
    <query>index=main | stats count by host</query>
    <earliest>-24h</earliest>
    <latest>now</latest>
  </search>
  
  <row>
    <panel>
      <title>Event Count by Host</title>
      <chart>
        <search base="base_search">
          <query>| sort - count | head 10</query>
        </search>
        <option name="charting.chart">bar</option>
        <option name="charting.drilldown">all</option>
      </chart>
    </panel>
  </row>
  
  <row>
    <panel>
      <table>
        <title>Recent Events</title>
        <search>
          <query>index=main | head 100 | table _time, host, source, _raw</query>
        </search>
        <option name="count">20</option>
        <option name="drilldown">row</option>
      </table>
    </panel>
  </row>
</dashboard>
```

### Visualization Types

| Element | Description |
|---------|-------------|
| `<chart>` | Line, bar, area, pie, scatter charts |
| `<table>` | Tabular data display |
| `<single>` | Single value with trend |
| `<map>` | Geographic visualization |
| `<event>` | Raw event listing |
| `<html>` | Custom HTML content |

### Chart Options

```xml
<chart>
  <option name="charting.chart">line</option>
  <option name="charting.chart.stackMode">stacked</option>
  <option name="charting.axisTitleX.text">Time</option>
  <option name="charting.axisTitleY.text">Count</option>
  <option name="charting.legend.placement">bottom</option>
  <option name="charting.drilldown">all</option>
</chart>
```

**Chart Types:** `line`, `area`, `bar`, `column`, `pie`, `scatter`, `bubble`, `radialGauge`, `fillerGauge`, `markerGauge`

### Form Inputs

```xml
<form version="1.1">
  <label>My Form</label>
  
  <fieldset submitButton="true" autoRun="true">
    <input type="time" token="time_range">
      <label>Time Range</label>
      <default>
        <earliest>-24h</earliest>
        <latest>now</latest>
      </default>
    </input>
    
    <input type="dropdown" token="selected_host">
      <label>Host</label>
      <choice value="*">All</choice>
      <search>
        <query>| metadata type=hosts index=main | table host</query>
      </search>
      <fieldForLabel>host</fieldForLabel>
      <fieldForValue>host</fieldForValue>
      <default>*</default>
    </input>
    
    <input type="text" token="search_term">
      <label>Search</label>
      <default>*</default>
    </input>
  </fieldset>
  
  <row>
    <panel>
      <chart>
        <search>
          <query>index=main host=$selected_host$ $search_term$ | timechart count</query>
          <earliest>$time_range.earliest$</earliest>
          <latest>$time_range.latest$</latest>
        </search>
      </chart>
    </panel>
  </row>
</form>
```

### Dashboard Studio (JSON)

```json
{
  "dataSources": {
    "ds_main": {
      "type": "ds.search",
      "options": {
        "query": "index=main | stats count by host",
        "queryParameters": {
          "earliest": "-24h",
          "latest": "now"
        }
      }
    }
  },
  "visualizations": {
    "viz_chart": {
      "type": "splunk.column",
      "dataSources": {
        "primary": "ds_main"
      },
      "options": {
        "xAxisTitleText": "Host",
        "yAxisTitleText": "Count"
      }
    }
  },
  "layout": {
    "type": "absolute",
    "options": {
      "width": 1200,
      "height": 800
    },
    "structure": [
      {
        "item": "viz_chart",
        "position": {"x": 0, "y": 0, "w": 600, "h": 400}
      }
    ]
  }
}
```

---

## Common Admin Tasks

### Index Management

```spl
# Check index sizes
| rest /services/data/indexes 
| table title, currentDBSizeMB, totalEventCount, maxTime

# View data distribution by sourcetype
| metadata type=sourcetypes index=main 
| table sourcetype, totalCount, recentTime

# Check license usage
index=_internal source=*license_usage.log* type=Usage
| stats sum(b) as bytes by idx
| eval GB = round(bytes/1024/1024/1024, 2)
```

### User and Role Management

```spl
# List users
| rest /services/authentication/users
| table title, roles, email

# Check role capabilities
| rest /services/authorization/roles
| mvexpand capabilities
| stats values(capabilities) by title
```

### Health Monitoring

```spl
# Splunk internal errors
index=_internal sourcetype=splunkd log_level=ERROR
| stats count by component, message

# Search performance
index=_audit action=search
| stats avg(total_run_time) as avg_time, count by user, search
| where avg_time > 60

# Forwarder status
| rest /services/deployment/server/clients
| table hostname, lastPhoneHomeTime, build
```

### Troubleshooting

**Check configuration:**
```bash
# List effective configuration
$SPLUNK_HOME/bin/splunk btool inputs list --debug
$SPLUNK_HOME/bin/splunk btool props list <sourcetype> --debug

# Check for errors
$SPLUNK_HOME/bin/splunk btool check
```

**Debug data parsing:**
```spl
# Check event breaking
index=main sourcetype=problematic earliest=-15m
| eval len=len(_raw)
| stats min(len), max(len), avg(len)

# Verify timestamp extraction
index=main sourcetype=problematic earliest=-15m
| eval extracted_time = strftime(_time, "%Y-%m-%d %H:%M:%S")
| table _raw, extracted_time
```
