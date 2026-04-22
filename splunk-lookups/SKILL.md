---
name: splunk-lookups
description: "Splunk Lookups reference for CSV file-based, KV Store, and automatic lookups. Use when: (1) Creating or configuring CSV lookup table files and transforms.conf definitions, (2) Setting up KV Store collections in collections.conf and KV Store lookup definitions in transforms.conf, (3) Configuring automatic lookups in props.conf with LOOKUP- directives, (4) Using lookup, inputlookup, and outputlookup SPL commands, (5) Troubleshooting lookup errors such as 'lookup table not found', missing fields, or auto-lookup failures, (6) Packaging apps with lookups for AppInspect and Splunk Cloud vetting."
---

# Splunk Lookups Reference

## Overview

Lookups enrich event data by adding field-value combinations from external tables. At search time, Splunk matches field values in your events to field values in a lookup table and appends corresponding output fields to those events.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        LOOKUP ARCHITECTURE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  LOOKUP TABLE FILES         LOOKUP DEFINITIONS        AUTOMATIC LOOKUPS     │
│  ─────────────────          ──────────────────        ─────────────────     │
│                                                                              │
│  ┌─────────────┐           ┌──────────────────┐     ┌──────────────────┐   │
│  │ CSV files   │           │ transforms.conf  │     │ props.conf       │   │
│  │ lookups/    │ ◄─────── │ [lookup_name]    │ ◄── │ LOOKUP-class =   │   │
│  │ *.csv       │ filename  │ filename = X.csv │     │ lookup_name ...  │   │
│  └─────────────┘           │ max_matches = 1  │     │ OUTPUT field1    │   │
│                            └──────────────────┘     └──────────────────┘   │
│  ┌─────────────┐           ┌──────────────────┐            │               │
│  │ KV Store    │           │ transforms.conf  │            │               │
│  │ collections │ ◄─────── │ [lookup_name]    │ ◄──────────┘               │
│  │ (MongoDB)   │ collection│ external_type =  │                             │
│  └─────────────┘           │   kvstore        │     ┌──────────────────┐   │
│       ▲                    │ collection = X   │     │ collections.conf │   │
│       │                    │ fields_list = .. │     │ [collection_name]│   │
│       └────────────────────┴──────────────────┘     │ field.X = type   │   │
│                                                      └──────────────────┘   │
│                                                                              │
│  SEARCH-TIME ORDER: extractions → aliases → calc fields → LOOKUPS → ...    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

Lookups are **seventh** in the search-time operations sequence, processed after calculated fields but before event types and tags.

---

## Lookup Types

| Type | Data Source | Config File | Use When |
|------|------------|-------------|----------|
| **CSV** | Static CSV file in `lookups/` | `transforms.conf` | Small, static reference data (< 10 MB) |
| **KV Store** | MongoDB-backed collection | `transforms.conf` + `collections.conf` | Large, frequently updated, or API-managed data |
| **External** | Python script or binary | `transforms.conf` | Dynamic lookups (DNS, API calls) |
| **Geospatial** | KMZ/KML file in `lookups/` | `transforms.conf` | Choropleth maps, geographic feature matching |

### Decision Framework: CSV vs KV Store

| Criterion | CSV Lookup | KV Store Lookup |
|-----------|-----------|----------------|
| Data size | Small (< 10 MB recommended) | Large (millions of rows) |
| Update frequency | Infrequent (manual CSV replacement) | Frequent (REST API, outputlookup, UI) |
| Replication | Replicated via knowledge bundles to indexers automatically | Lives on search head; requires `replicate = true` in `collections.conf` for indexer distribution |
| CRUD operations | Replace entire file | Insert, update, delete individual records |
| Field types | All strings | Typed fields (string, number, bool, time, cidr) |
| Search commands | `lookup`, `inputlookup`, `outputlookup` | `lookup`, `inputlookup`, `outputlookup`, REST API |
| Multivalue fields | Not natively supported | Supported via `array` type |
| AppInspect | CSV file must be in `lookups/` directory | Collection defined in `collections.conf` |

---

## CSV Lookups — Complete Configuration

### Step 1: Create the CSV File

Place the CSV file in the app's `lookups/` directory:

```
<app_id>/
├── lookups/
│   ├── asset_metadata.csv
│   ├── severity_levels.csv
│   └── threshold_values.csv
```

**CSV file requirements:**
- First row MUST be a header row with field names
- Header row MUST NOT exceed 4096 characters
- Use Unix-style line endings (LF) or Windows-style (CRLF)
- Pre-OS X Macintosh-style line endings (CR only) are NOT supported
- UTF-8 encoding required
- Quote fields containing commas, newlines, or double quotes

**Example CSV — `asset_metadata.csv`:**
```csv
device_id,asset_name,location,zone,criticality,owner
PLK-001,Packaging Line PLC,Building A Floor 2,Zone 1,high,ops_team_a
PLK-002,Assembly Line PLC,Building B Floor 1,Zone 2,critical,ops_team_b
HMI-001,Control Room HMI,Building A Floor 1,Zone 1,high,ops_team_a
RTU-001,Remote Terminal Unit,Substation 4,Zone 3,critical,field_team
```

### Step 2: Define the Lookup in transforms.conf

```ini
[asset_metadata_lookup]
filename = asset_metadata.csv
max_matches = 1
min_matches = 0
default_match = unknown
case_sensitive_match = false
```

**All CSV lookup attributes in transforms.conf:**

| Attribute | Required | Default | Description |
|-----------|----------|---------|-------------|
| `filename` | **Yes** | — | Name of CSV file in `lookups/` directory. Path components are stripped — use filename only. |
| `max_matches` | No | `100` (no time field) or `1` (with time field) | Maximum matches per input value (range: 1–1000) |
| `min_matches` | No | `0` | Minimum matches; if fewer found, `default_match` fills the gap |
| `default_match` | No | empty string | Value returned when fewer than `min_matches` found |
| `case_sensitive_match` | No | `true` | Set `false` for case-insensitive matching |
| `match_type` | No | `EXACT` | `WILDCARD(<field>)` or `CIDR(<field>)` for non-exact matching |
| `filter` | No | — | Boolean expression to prefilter large CSV tables |
| `check_permission` | No | `false` | When `true`, verifies user permissions for `outputlookup` writes |
| `replicate` | No | `true` | When `false`, CSV is not replicated to indexers (use for `inputcsv`/`outputcsv` only) |
| `index_fields_list` | No | all fields | Comma-separated list of fields to index for faster matching |
| `time_field` | No | — | Field name containing time values for time-bounded lookups |
| `time_format` | No | `%s.%Q` | `strptime` format of the `time_field` values |

### Step 3: Make It Automatic (Optional but Common)

Add a `LOOKUP-` directive in `props.conf` under the relevant sourcetype stanza:

```ini
[my_sourcetype]
LOOKUP-asset_enrich = asset_metadata_lookup device_id OUTPUTNEW asset_name, location, zone, criticality, owner
```

---

## KV Store Lookups — Complete Configuration

KV Store lookups require THREE configuration files working together.

### Step 1: Define the Collection in collections.conf

```ini
[asset_inventory]
field.device_id = string
field.asset_name = string
field.location = string
field.zone = string
field.criticality = string
field.owner = string
field.last_updated = time
field.ip_address = cidr
replicate = false
```

**collections.conf field types:**

| Type | Description | Example Values |
|------|-------------|----------------|
| `string` | Text value (default if no type specified) | `"PLC-001"`, `"Building A"` |
| `number` | Numeric value (integer or float) | `42`, `98.6` |
| `bool` | Boolean value | `true`, `false` |
| `time` | Epoch timestamp | `1679529600` |
| `cidr` | IPv4/IPv6 CIDR address | `"192.168.1.0/24"` |
| `array` | Multi-value field (JSON array) | Not declared — use `accelerated_fields` |

**collections.conf key attributes:**

| Attribute | Default | Description |
|-----------|---------|-------------|
| `field.<name>` | — | Defines a field and its type |
| `replicate` | `false` | Set `true` to replicate collection to indexers for auto-lookups |
| `accelerated_fields.<name>` | — | JSON object defining compound indexes for query performance |
| `profilingEnabled` | `false` | Enable query profiling for debugging |
| `profilingThresholdMs` | `1000` | Log queries slower than this threshold |
| `enforceTypes` | `false` | Reject records that don't match declared field types |

**CRITICAL: `replicate` for Automatic Lookups**

KV Store collections live on the search head by default. If you want to use a KV Store lookup as an **automatic lookup** (search-time enrichment via `LOOKUP-` in `props.conf`), the lookup runs on the search head, NOT on the indexers. This is fine for most use cases.

However, if you need the auto-lookup to run on indexers (distributed search), set `replicate = true` in `collections.conf`. Without it, indexers will report `"lookup table not found"` errors.

### Step 2: Define the Lookup in transforms.conf

```ini
[asset_inventory_lookup]
external_type = kvstore
collection = asset_inventory
fields_list = _key, device_id, asset_name, location, zone, criticality, owner, last_updated, ip_address
case_sensitive_match = false
```

**All KV Store lookup attributes in transforms.conf:**

| Attribute | Required | Default | Description |
|-----------|----------|---------|-------------|
| `external_type` | **Yes** | — | MUST be `kvstore` |
| `collection` | **Yes** | — | Name of the KV Store collection from `collections.conf` |
| `fields_list` | **Yes** | — | Comma-and-space-separated list of fields. Include `_key` to enable record-level updates. |
| `case_sensitive_match` | No | `true` | Set `false` for case-insensitive matching |
| `filter` | No | — | Boolean expression to prefilter large collections |
| `max_matches` | No | `100` | Maximum matches per input value |
| `min_matches` | No | `0` | Minimum matches |
| `default_match` | No | empty string | Fallback value |
| `match_type` | No | `EXACT` | `WILDCARD(<field>)` or `CIDR(<field>)` |

### Step 3: Make It Automatic (Optional)

```ini
[my_sourcetype]
LOOKUP-asset_inventory = asset_inventory_lookup device_id OUTPUTNEW asset_name, location, zone, criticality, owner
```

### Step 4: Populate the Collection

KV Store collections start empty. Populate them using one of:

**SPL with outputlookup:**
```spl
| inputlookup asset_metadata.csv
| outputlookup asset_inventory_lookup
```

**REST API (batch insert):**
```bash
curl -k -u admin:password \
  https://localhost:8089/servicesNS/nobody/my_app/storage/collections/data/asset_inventory/batch_save \
  -H "Content-Type: application/json" \
  -d '[{"device_id":"PLK-001","asset_name":"Packaging Line PLC","location":"Building A"}]'
```

**REST API (single record):**
```bash
curl -k -u admin:password \
  https://localhost:8089/servicesNS/nobody/my_app/storage/collections/data/asset_inventory \
  -H "Content-Type: application/json" \
  -d '{"device_id":"PLK-001","asset_name":"Packaging Line PLC"}'
```

---

## Automatic Lookups — props.conf Syntax

Automatic lookups run at search time without requiring the `lookup` command in SPL. They are defined in `props.conf`.

### Syntax

```ini
[<sourcetype>]
LOOKUP-<class> = <lookup_name> <match_field_in_table> [AS <match_field_in_event>] OUTPUT|OUTPUTNEW <output_field> [AS <output_field_in_event>] [, <output_field2> [AS <alias2>]]
```

### Key Rules

1. **`<class>` MUST be unique** within a sourcetype stanza. Two `LOOKUP-` directives with the same class name will conflict — the last one wins.
2. **Use `OUTPUTNEW` (not `OUTPUT`)** to avoid overwriting fields that already exist in events. `OUTPUT` overwrites existing field values; `OUTPUTNEW` only adds fields that are not already present.
3. **Use `AS` for field renaming** when the lookup table field name differs from the desired event field name, or when the match field name differs between event and table.
4. **Omitting OUTPUT/OUTPUTNEW** causes ALL fields from the lookup table to be added to events — this is rarely desirable and can cause field name collisions.
5. **Nested automatic lookups are NOT supported.** Splunk does not guarantee execution order between multiple `LOOKUP-` directives on the same sourcetype. Do not rely on one auto-lookup's output as the input for another.
6. **Lookups cannot reference event types or tags** — only fields from extractions, aliases, and calculated fields.

### Complete Example

```ini
[industrial:sensor]
# Great 8 parsing (abbreviated)
SHOULD_LINEMERGE = false
TIME_FORMAT = %Y-%m-%dT%H:%M:%S
TIME_PREFIX = timestamp=

# Field aliases
FIELDALIAS-src = source_ip AS src

# Calculated fields
EVAL-vendor_product = "Acme Industrial Sensors"

# Automatic lookups (processed AFTER aliases and calculated fields)
LOOKUP-asset_enrich = asset_metadata_lookup device_id OUTPUTNEW asset_name, location, zone, criticality
LOOKUP-threshold = threshold_values_lookup sensor_type OUTPUTNEW warn_low, warn_high, alarm_low, alarm_high
```

### CRITICAL: Avoid Auto-Lookup Reference Cycles

A reference cycle occurs when a lookup's match field and output field overlap, either within one lookup or across lookups on the same sourcetype.

```ini
# BAD — reference cycle: 'status' is both match and implicit output
LOOKUP-bad = my_lookup status

# GOOD — explicit OUTPUT prevents cycle
LOOKUP-good = my_lookup status OUTPUT status_description, status_category
```

---

## Field Matching Rules

### match_type — WILDCARD and CIDR

Use `match_type` for non-exact matching. The field specified in `match_type` is the **lookup table field**, not the event field.

**WILDCARD matching:**
```ini
[device_pattern_lookup]
filename = device_patterns.csv
match_type = WILDCARD(device_pattern)
```

The CSV file contains wildcard patterns:
```csv
device_pattern,device_class,priority
PLC-*,plc,high
HMI-*,hmi,medium
RTU-*,rtu,critical
*-SPARE,spare,low
```

**CIDR matching (IPv4 and IPv6):**
```ini
[subnet_lookup]
filename = network_subnets.csv
match_type = CIDR(subnet)
```

The CSV file contains CIDR notation:
```csv
subnet,zone_name,security_level
192.168.1.0/24,OT Zone 1,high
10.0.0.0/8,IT Network,medium
172.16.0.0/16,DMZ,critical
2001:db8::/32,IPv6 Internal,high
```

### Combining match types

```ini
[complex_lookup]
filename = complex_patterns.csv
match_type = WILDCARD(host_pattern), CIDR(network)
```

---

## SPL Commands for Lookups

### lookup — Add Fields to Events

```spl
... | lookup asset_metadata_lookup device_id OUTPUT asset_name, location
... | lookup asset_metadata_lookup device_id AS src_device OUTPUT asset_name AS src_asset_name
```

### inputlookup — Search Lookup Contents

```spl
| inputlookup asset_metadata.csv
| inputlookup asset_metadata.csv WHERE criticality="critical"
| inputlookup asset_inventory_lookup WHERE device_id="PLK-*"
| inputlookup append=true asset_metadata.csv
```

### outputlookup — Write to Lookup Tables

```spl
... | outputlookup asset_metadata.csv
... | outputlookup asset_inventory_lookup
... | outputlookup createinapp=true append=true asset_inventory_lookup
```

**outputlookup key behaviors:**
- Without `append=true`, the existing table is replaced entirely
- With `append=true`, new records are added (KV Store: upsert by `_key`)
- `create_empty=false` prevents creating empty lookup files
- `createinapp=true` creates the file in the current app context

### Populating KV Store from CSV

```spl
| inputlookup asset_metadata.csv
| outputlookup asset_inventory_lookup
```

### Populating CSV from Search Results

```spl
index=assets sourcetype=asset_scan
| stats latest(asset_name) AS asset_name, latest(location) AS location BY device_id
| outputlookup asset_metadata.csv
```

---

## App Packaging — Lookup File Layout

### Required Directory Structure

```
<app_id>/
├── default/
│   ├── transforms.conf        # Lookup definitions
│   ├── props.conf             # Automatic lookup directives (LOOKUP-)
│   └── collections.conf       # KV Store collection definitions (if using KV Store)
├── lookups/
│   ├── asset_metadata.csv     # CSV lookup table files
│   ├── severity_levels.csv
│   └── threshold_values.csv
└── metadata/
    └── default.meta           # Permissions for lookups
```

### metadata/default.meta — Lookup Permissions

```ini
[]
access = read : [ * ], write : [ admin, power ]
export = system

[lookups]
access = read : [ * ], write : [ admin, power ]
export = system

[transforms/asset_metadata_lookup]
export = system
```

### AppInspect Checklist for Lookups

| Check | Requirement |
|-------|-------------|
| CSV files location | MUST be in `lookups/` directory, NOT in `default/` or `local/` |
| CSV file encoding | MUST be UTF-8 |
| CSV header row | MUST exist and be < 4096 characters |
| transforms.conf `filename` | MUST be filename only — no paths. Splunk strips path separators. |
| KV Store `collections.conf` | MUST exist if `external_type = kvstore` is used in transforms.conf |
| No `local/` directory | CSV files in `local/lookups/` will fail Cloud vetting |
| File permissions | 644 for files, 755 for directories |
| No macOS artifacts | No `.DS_Store` or `._*` files in `lookups/` directory |

---

## Common Errors and Troubleshooting

### "Lookup table not found"

| Cause | Fix |
|-------|-----|
| CSV file not in `lookups/` directory | Move CSV to `<app>/lookups/` |
| `filename` has a path component | Use filename only: `filename = my_lookup.csv` |
| KV Store collection not defined | Create `collections.conf` with `[collection_name]` stanza |
| KV Store auto-lookup on indexer without replication | Set `replicate = true` in `collections.conf` |
| App not visible or exported | Check `metadata/default.meta` has `export = system` |
| Stanza name mismatch | Verify `LOOKUP-` in props.conf references exact stanza name from transforms.conf |

### "Unknown lookup field"

| Cause | Fix |
|-------|-----|
| Field not in CSV header row | Add field to CSV header |
| Field not in `fields_list` (KV Store) | Add field to `fields_list` in transforms.conf |
| Typo in field name | Check exact spelling in lookup definition vs `LOOKUP-` directive |

### Auto-Lookup Fields Not Appearing

| Cause | Fix |
|-------|-----|
| `LOOKUP-` class name collision | Use unique class names: `LOOKUP-asset` and `LOOKUP-threshold`, not two `LOOKUP-enrich` |
| Missing `OUTPUT`/`OUTPUTNEW` clause | Add explicit output fields to prevent reference cycles |
| Lookup processes after needed field | Lookups cannot reference event types/tags — only extractions, aliases, calc fields |
| Splunk not restarted after .conf change | Restart Splunk or use `| debug refresh` |
| KV Store collection empty | Populate with `outputlookup` or REST API |

### Auto-Lookup Search Errors at Search Time

| Error | Cause | Fix |
|-------|-------|-----|
| `Lookup table 'X' does not exist or is inaccessible` | Missing transforms.conf stanza or CSV file | Verify stanza name matches, CSV exists in lookups/ |
| `Unknown search command 'lookup'` | lookup command disabled | Check `limits.conf [lookup]` settings |
| `Too many lookup results` | Exceeded `max_matches` or result limit | Increase `max_matches` or add `filter` to reduce rows |
| `Subsearch is not supported in automatic lookups` | Using pipe/subsearch in eventtype referenced by lookup | Lookups process before eventtypes — restructure logic |
| `Lookup file exceeds maximum size` | CSV file too large for bundle replication | Use KV Store instead, or increase `limits.conf [lookup] max_memtable_bytes` |

---

## Complete Working Example — CSV Lookup

### 1. Create the CSV file: `lookups/http_status.csv`

```csv
status,status_description,status_type
200,OK,Successful
301,Moved Permanently,Redirection
302,Found,Redirection
400,Bad Request,Client Error
401,Unauthorized,Client Error
403,Forbidden,Client Error
404,Not Found,Client Error
500,Internal Server Error,Server Error
502,Bad Gateway,Server Error
503,Service Unavailable,Server Error
```

### 2. Define in transforms.conf

```ini
[http_status_lookup]
filename = http_status.csv
max_matches = 1
default_match = Unknown
case_sensitive_match = false
```

### 3. Make automatic in props.conf

```ini
[access_combined]
LOOKUP-http_status = http_status_lookup status OUTPUTNEW status_description, status_type
```

### 4. Use in SPL (manual invocation)

```spl
index=web sourcetype=access_combined
| lookup http_status_lookup status OUTPUT status_description, status_type
| stats count BY status, status_description, status_type
```

---

## Complete Working Example — KV Store Lookup

### 1. Define the collection in collections.conf

```ini
[asset_inventory]
field.device_id = string
field.asset_name = string
field.location = string
field.zone = string
field.criticality = string
field.owner = string
field.ip_address = string
field.last_maintenance = time
replicate = false
enforceTypes = false
```

### 2. Define the lookup in transforms.conf

```ini
[asset_inventory_lookup]
external_type = kvstore
collection = asset_inventory
fields_list = _key, device_id, asset_name, location, zone, criticality, owner, ip_address, last_maintenance
case_sensitive_match = false
max_matches = 1
```

### 3. Make automatic in props.conf

```ini
[industrial:sensor]
LOOKUP-asset = asset_inventory_lookup device_id OUTPUTNEW asset_name, location, zone, criticality, owner
```

### 4. Populate the KV Store

```spl
| makeresults
| eval device_id="PLK-001", asset_name="Packaging Line PLC", location="Building A", zone="Zone 1", criticality="high", owner="ops_team_a"
| outputlookup asset_inventory_lookup
```

Or from a CSV seed file:
```spl
| inputlookup asset_seed_data.csv
| outputlookup asset_inventory_lookup
```

### 5. Query the KV Store

```spl
| inputlookup asset_inventory_lookup
| search criticality="critical"
| table device_id, asset_name, location, zone

| inputlookup asset_inventory_lookup WHERE criticality="critical"
```

### 6. Update individual records (via REST API)

```bash
# Update a specific record by _key
curl -k -u admin:password \
  https://localhost:8089/servicesNS/nobody/my_app/storage/collections/data/asset_inventory/RECORD_KEY \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{"device_id":"PLK-001","asset_name":"Packaging Line PLC v2","location":"Building A"}'

# Delete a specific record
curl -k -u admin:password \
  https://localhost:8089/servicesNS/nobody/my_app/storage/collections/data/asset_inventory/RECORD_KEY \
  -X DELETE
```

---

## Complete Working Example — Two-Stage Enrichment

A common OT/IoT pattern: first enrich with asset metadata, then apply thresholds based on an enriched field.

**IMPORTANT:** Splunk does NOT support nested automatic lookups. For two-stage enrichment, use one of these patterns:

### Pattern A: Single SPL Query (Recommended)

```spl
index=ot sourcetype=industrial:sensor
| lookup asset_metadata_lookup device_id OUTPUT asset_name, sensor_type
| lookup threshold_values_lookup sensor_type OUTPUT warn_high, alarm_high
| eval status=case(value > alarm_high, "critical", value > warn_high, "warning", true(), "normal")
```

### Pattern B: Pre-Merged Lookup Table

Create a single merged CSV that contains both asset metadata AND thresholds:

```csv
device_id,asset_name,location,sensor_type,warn_high,alarm_high,warn_low,alarm_low
PLK-001,Packaging PLC,Building A,vibration,4.5,7.0,0.5,0.1
PLK-002,Assembly PLC,Building B,temperature,35,45,5,0
```

```ini
# transforms.conf
[device_enrichment_lookup]
filename = device_enrichment.csv
max_matches = 1
case_sensitive_match = false

# props.conf — single auto-lookup provides all enrichment
[industrial:sensor]
LOOKUP-device = device_enrichment_lookup device_id OUTPUTNEW asset_name, location, sensor_type, warn_high, alarm_high, warn_low, alarm_low
```

### Pattern C: Scheduled Search to Build Merged Lookup

```spl
| inputlookup asset_metadata.csv
| lookup threshold_values_lookup sensor_type OUTPUT warn_high, alarm_high, warn_low, alarm_low
| outputlookup device_enrichment.csv
```

Schedule this search to run daily to keep the merged lookup current.

---

## Time-Based Lookups

Time-based lookups match events to lookup rows based on time proximity.

```ini
[maintenance_schedule_lookup]
filename = maintenance_schedule.csv
time_field = scheduled_time
time_format = %Y-%m-%dT%H:%M:%S
max_matches = 1
```

The CSV file:
```csv
device_id,scheduled_time,maintenance_type,technician
PLK-001,2026-03-15T08:00:00,preventive,John Smith
PLK-001,2026-04-15T08:00:00,preventive,Jane Doe
PLK-002,2026-03-20T14:00:00,corrective,Bob Wilson
```

When `time_field` is specified, `max_matches` defaults to `1` and matches are returned in descending time order (most recent first).

---

## Performance Optimization

### CSV Lookups

1. **Keep CSV files small** — under 10 MB. For larger datasets, use KV Store.
2. **Use `index_fields_list`** to index only the fields used for matching:
   ```ini
   [large_lookup]
   filename = large_reference.csv
   index_fields_list = device_id, ip_address
   ```
3. **Use `filter`** to prefilter when only a subset of rows is needed:
   ```ini
   [filtered_lookup]
   filename = all_assets.csv
   filter = (criticality="critical") OR (criticality="high")
   ```
4. **Set `replicate = false`** for lookups only used with `inputcsv`/`outputcsv` on the search head.

### KV Store Lookups

1. **Add accelerated fields** for frequently queried fields:
   ```ini
   [asset_inventory]
   field.device_id = string
   field.zone = string
   accelerated_fields.device = {"device_id": 1}
   accelerated_fields.zone = {"zone": 1}
   ```
2. **Use `filter`** to restrict the collection scan:
   ```ini
   [asset_lookup]
   external_type = kvstore
   collection = asset_inventory
   fields_list = _key, device_id, asset_name, zone
   filter = (zone="Zone 1") OR (zone="Zone 2")
   ```
3. **Use typed fields** in `collections.conf` for proper comparison and sorting.

---

## Splunk Cloud Considerations

| Topic | Requirement |
|-------|-------------|
| CSV file upload | Upload via Settings > Lookups > Lookup Table Files, or package in app |
| KV Store access | Available; manage via REST API or SPL |
| collections.conf | Include in app package; cannot edit on server directly |
| replicate | Set `false` for KV Store collections on Cloud (indexers are managed) |
| lookup file size limit | 500 MB default; check with Splunk Cloud admin |
| outputlookup permissions | Requires `check_permission = true` and correct role capabilities |

---

## Anti-Patterns — NEVER Do These

| Anti-Pattern | Why It Fails | Correct Approach |
|-------------|-------------|-----------------|
| Put CSV files in `default/` instead of `lookups/` | Splunk only searches `lookups/` directory for CSV files | Always use `<app>/lookups/` |
| Use `filename = /opt/splunk/etc/apps/myapp/lookups/data.csv` | Path components are stripped; only filename is used | Use `filename = data.csv` |
| Define `external_type = kvstore` without `collections.conf` | Collection doesn't exist; "lookup table not found" error | Always create matching `collections.conf` stanza |
| Use two `LOOKUP-` directives with the same class name | Second one silently overwrites the first | Use unique class names: `LOOKUP-asset`, `LOOKUP-threshold` |
| Rely on nested automatic lookups | Splunk does not support chained auto-lookups | Use merged lookup table or explicit `lookup` commands in SPL |
| Omit `OUTPUT`/`OUTPUTNEW` in auto-lookup | All table fields added to events, causing field collisions and reference cycles | Always specify explicit output fields |
| Use `OUTPUT` when `OUTPUTNEW` is appropriate | Overwrites existing event fields with lookup values | Use `OUTPUTNEW` unless intentional overwrite is needed |
| Create KV Store auto-lookup without `replicate = true` on indexers | Indexers cannot find the collection | Set `replicate = true` in `collections.conf`, or accept search-head-only lookup |
| Leave KV Store collection empty after defining lookup | Lookup matches nothing; appears broken | Populate via `outputlookup`, REST API, or seed data |
| Use `match_type = WILDCARD` on event field instead of lookup table field | WILDCARD applies to the lookup table column, not the event data | The field named in `WILDCARD()` must be the lookup table column containing patterns |

---

## Validation Queries

### Verify CSV Lookup Works

```spl
| inputlookup asset_metadata.csv
| stats count
| eval status=if(count>0, "OK: ".count." rows", "EMPTY: lookup has no data")
```

### Verify KV Store Collection Has Data

```spl
| inputlookup asset_inventory_lookup
| stats count
| eval status=if(count>0, "OK: ".count." records", "EMPTY: populate the KV Store collection")
```

### Verify Auto-Lookup Adds Fields

```spl
index=myindex sourcetype=my_sourcetype
| head 10
| fields device_id, asset_name, location
| where isnotnull(asset_name)
| stats count AS enriched_events
```

### Check for Lookup Errors in Internal Logs

```spl
index=_internal sourcetype=splunkd component=LookupProcessor log_level=ERROR
| stats count BY message
| sort -count
```

### Verify Lookup Definition Exists

```spl
| rest /servicesNS/-/-/data/transforms/lookups
| search title="asset_metadata_lookup"
| table title, type, filename, collection
```

### Verify KV Store Collection Schema

```spl
| rest /servicesNS/-/-/storage/collections/config
| search title="asset_inventory"
| table title, field.*
```

---

## References

- [About Lookups — Splunk Docs](https://help.splunk.com/en/splunk-enterprise/manage-knowledge-objects/knowledge-management-manual/9.2/use-lookups-in-splunk-web/about-lookups)
- [Configure CSV Lookups](https://help.splunk.com/en/splunk-enterprise/manage-knowledge-objects/knowledge-management-manual/9.2/use-the-configuration-files-to-configure-lookups/configure-csv-lookups)
- [Configure KV Store Lookups](https://help.splunk.com/en/splunk-enterprise/manage-knowledge-objects/knowledge-management-manual/9.2/use-the-configuration-files-to-configure-lookups/configure-kv-store-lookups)
- [Make Your Lookup Automatic](https://help.splunk.com/en/splunk-enterprise/manage-knowledge-objects/knowledge-management-manual/9.2/use-the-configuration-files-to-configure-lookups/make-your-lookup-automatic)
- [Field Matching Rules](https://help.splunk.com/en/splunk-enterprise/manage-knowledge-objects/knowledge-management-manual/9.2/use-the-configuration-files-to-configure-lookups/add-field-matching-rules-to-your-lookup-configuration)
- [transforms.conf Reference](https://docs.splunk.com/Documentation/Splunk/latest/Admin/Transformsconf)
- [collections.conf Reference](https://docs.splunk.com/Documentation/Splunk/latest/Admin/Collectionsconf)

---

## Related Skills

- **splunk-admin**: Core Splunk configuration, SPL, and knowledge objects
- **splunk-cim**: CIM field mapping and data model routing (uses lookups for enrichment)
- **splunk-app-dev**: App packaging, AppInspect, and Cloud vetting
- **splunk-spl-commands**: SPL command reference including lookup, inputlookup, outputlookup
