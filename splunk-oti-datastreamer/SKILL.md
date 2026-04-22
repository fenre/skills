---
name: splunk-oti-datastreamer
description: >
  OTI Datastreamer pipeline and HEC ingest configuration. Use this skill when:
  (1) Configuring HEC tokens and endpoints for OTI Cloud or Splunk Cloud ingest;
  (2) Setting up props.conf and transforms.conf for edge_hub_* index parsing;
  (3) Tuning HEC client settings — batch size, compression, retry logic;
  (4) Troubleshooting data pipeline issues — clock skew, dropped events, queue backup;
  (5) Understanding the edge collection → transport → indexing pipeline stages;
  (6) Configuring index-level tuning (maxHotBuckets, retention) for OT workloads.
---

# OTI Datastreamer Pipeline Configuration

## HEC Configuration

### Token Creation (Splunk Cloud)
1. Navigate to Settings → Data Inputs → HTTP Event Collector
2. Create new token with:
   - Name: `edge_hub_ingest`
   - Source type: `_json` or `edge_hub_ot`
   - Default index: `edge_hub_ot`
   - Allowed indexes: `edge_hub_*`

### HEC Endpoint URLs

**Standard Ingest:**
```
https://http-inputs-<stack>.splunkcloud.com/services/collector/event
```

**Raw Endpoint (single events):**
```
https://http-inputs-<stack>.splunkcloud.com/services/collector/raw
```

**Health Check:**
```
https://http-inputs-<stack>.splunkcloud.com/services/collector/health
```

### Request Headers
```http
Authorization: Splunk <HEC_TOKEN>
Content-Type: application/json
X-Splunk-Request-Channel: <CHANNEL_GUID>
```

### Payload Formats

**Single Event:**
```json
{
  "event": {
    "hub_name": "factory-hub-01",
    "temperature": 23.5,
    "humidity": 45.2
  },
  "time": 1769509500.123,
  "host": "ACT-076-1823-0086",
  "source": "edgehub/builtin/temperature/values",
  "sourcetype": "edge_hub_ot",
  "index": "edge_hub_ot"
}
```

**Batch Events (newline-delimited):**
```json
{"event": {"temp": 23.5}, "time": 1769509500.0, "index": "edge_hub_ot"}
{"event": {"temp": 23.6}, "time": 1769509501.0, "index": "edge_hub_ot"}
{"event": {"temp": 23.7}, "time": 1769509502.0, "index": "edge_hub_ot"}
```

### Acknowledgement Mode
Enable indexer acknowledgement for guaranteed delivery:
```http
X-Splunk-Request-Channel: <CHANNEL_GUID>
```

Check acknowledgement status:
```
GET /services/collector/ack?channel=<CHANNEL_GUID>
Body: {"acks": [0, 1, 2, 3]}
```

---

## Index Configuration

### props.conf
```ini
[edge_hub_ot]
INDEXED_EXTRACTIONS = json
KV_MODE = json
TIME_FORMAT = %s.%3N
TIME_PREFIX = "time":
MAX_TIMESTAMP_LOOKAHEAD = 20
TRUNCATE = 50000

[edge_hub_modbus]
INDEXED_EXTRACTIONS = json
KV_MODE = json
EVAL-value_numeric = tonumber(value)

[bms_json]
KV_MODE = json
TRANSFORMS-extract_payload = bms_payload_extract
```

### transforms.conf
```ini
[bms_payload_extract]
REGEX = "entity_id":"([^"]+)".*"state":"([^"]+)"
FORMAT = entity_id::$1 state::$2

[edge_hub_modbus_lookup]
filename = modbus_register_map.csv
```

### Calculated Fields (props.conf)
```ini
[edge_hub_ot]
# Convert Celsius to Fahrenheit
EVAL-temp_fahrenheit = if(isnotnull('edgehub.sensor.temperature'), ('edgehub.sensor.temperature' * 9/5) + 32, null())

# Classify IAQ readings
EVAL-iaq_category = case(
  'edgehub.sensor.iaq' <= 50, "Good",
  'edgehub.sensor.iaq' <= 100, "Moderate",
  'edgehub.sensor.iaq' <= 150, "Unhealthy Sensitive",
  'edgehub.sensor.iaq' <= 200, "Unhealthy",
  'edgehub.sensor.iaq' <= 300, "Very Unhealthy",
  true(), "Hazardous"
)
```

---

## Data Pipeline Stages

### 1. Edge Collection
```
[Edge Hub Device]
    │
    ├── Modbus Client ─→ JSON transform
    ├── MQTT Client ───→ JSON transform  
    ├── OPC UA Client ─→ JSON transform
    └── Built-in Sensors → JSON transform
                │
                ▼
         Local Buffer (SQLite)
```

### 2. Transport
```
         Local Buffer
              │
              ▼
    ┌─────────────────┐
    │   HEC Client    │
    │  - Batching     │
    │  - Compression  │
    │  - Retry logic  │
    └────────┬────────┘
             │
             ▼ HTTPS (port 8088)
    ┌─────────────────┐
    │  Load Balancer  │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │   HEC Endpoint  │
    │   (Indexers)    │
    └─────────────────┘
```

### 3. Indexing
```
    HEC Endpoint
         │
         ▼
    ┌─────────────────┐
    │  Parsing Queue  │
    │  - Time extract │
    │  - Field extract│
    │  - Transforms   │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │     Index       │
    │  (edge_hub_*)   │
    └─────────────────┘
```

---

## Performance Tuning

### HEC Client Settings
| Parameter | Description | Recommended |
|-----------|-------------|-------------|
| `batch_size` | Events per request | 100-1000 |
| `batch_timeout` | Max wait before send | 1-5 seconds |
| `max_retries` | Retry attempts | 3-5 |
| `retry_backoff` | Exponential backoff | 1s, 2s, 4s |
| `compression` | Enable gzip | Yes |

### Index Tuning
| Setting | Description | OT Workload |
|---------|-------------|-------------|
| `maxHotBuckets` | Concurrent writes | 10-20 |
| `maxDataSize` | Bucket size | 750MB |
| `frozenTimePeriodInSecs` | Retention | 86400 * 90 |

### Search Optimization
```spl
# Use tstats for high-volume metrics
| tstats count where index=edge_hub_ot by _time, host span=1m

# Accelerate common aggregations with summary indexing
index=summary report=edge_hub_hourly_stats
```

---

## Error Handling

### HEC Response Codes
| Code | Meaning | Action |
|------|---------|--------|
| 200 | Success | None |
| 400 | Bad request | Check payload format |
| 401 | Unauthorized | Verify token |
| 403 | Forbidden | Check token permissions |
| 503 | Service unavailable | Retry with backoff |

### Common Issues

**Clock Skew:**
```spl
index=edge_hub_* 
| eval time_drift = _indextime - _time
| where abs(time_drift) > 60
| stats count by host, avg(time_drift)
```

**Dropped Events:**
```spl
index=_internal sourcetype=splunkd component=HttpInputDataHandler
| search "dropped" OR "rejected"
| stats count by reason
```

**Queue Backup:**
```spl
index=_internal sourcetype=splunkd component=Metrics 
| search group=queue name=indexqueue
| timechart avg(current_size) by host
```
