---
name: splunk-edge-hub
description: "Splunk Edge Hub and OTI Cloud data ingestion guide. Use when working with: (1) Edge Hub source configurations for industrial protocols (Modbus TCP, OPC UA, MQTT, SNMP, BACnet), (2) OTI Datastreamer ingest methods and pipeline configurations, (3) Troubleshooting Edge Hub connectivity and data flow, (4) Creating dashboards or alerts for OT/IoT sensor data, (5) Understanding Edge Hub data schemas and sourcetypes"
---

# Splunk Edge Hub & OTI Datastreamer Ingestion

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA SOURCES                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  Industrial Devices    │  IoT Sensors        │  Network Equipment           │
│  • PLCs (Modbus TCP)   │  • Environmental    │  • Cisco Meraki              │
│  • OPC UA Servers      │  • Motion/Presence  │  • SNMP Devices              │
│  • BACnet Controllers  │  • MQTT Brokers     │  • Zeek Network Logs         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SPLUNK EDGE HUB                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   Modbus    │  │   OPC UA    │  │    MQTT     │  │    SNMP     │        │
│  │   Client    │  │   Client    │  │   Client    │  │   Poller    │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│         │                │                │                │                │
│         └────────────────┴────────────────┴────────────────┘                │
│                                    │                                         │
│                          Local Processing                                    │
│                    (JSON transformation, filtering)                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      OTI DATASTREAMER INGEST                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Ingest Pipeline: Edge Hub → HEC Endpoint → Indexers → Search Head  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  Ingest Methods:                                                             │
│  • HTTP Event Collector (HEC) - Primary method for Edge Hub                  │
│  • Splunk Connect for Kafka - High-volume streaming                          │
│  • Heavy Forwarder - On-prem aggregation                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SPLUNK OTI CLOUD                                      │
│  Indexes: edge_hub_ot, edge_hub_modbus, edge_hub_mqtt, edge_hub_opcua,      │
│           edge_hub_snmp, edge_hub_logs, edge_hub_zeek, bms                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Edge Hub Source Configurations

### Modbus TCP

Collects register data from PLCs and industrial controllers.

**Index:** `edge_hub_modbus`  
**Source:** `edgehub/modbus`  
**Sourcetype:** `_json`

**Data Schema:**
```json
{
  "hub_name": "Simon-test",
  "captureHostname": "192.168.0.97",
  "destPort": 5020,
  "deviceName": "rtu",
  "deviceId": 1,
  "startingAddress": 7144,
  "endingAddress": 7145,
  "unitIdentifier": "UINT_32BIT",
  "value": "470310",
  "functionCode": "READ_HOLDING_REGISTERS"
}
```

**Key Fields:**
| Field | Description |
|-------|-------------|
| `hub_name` | Edge Hub device identifier |
| `captureHostname` | Target PLC/device IP address |
| `destPort` | Modbus TCP port (default 502) |
| `deviceName` | Logical device name |
| `startingAddress` / `endingAddress` | Register address range |
| `unitIdentifier` | Data type (UINT_16BIT, UINT_32BIT, FLOAT_32BIT, etc.) |
| `value` | Raw register value |
| `functionCode` | Modbus function (READ_HOLDING_REGISTERS, READ_INPUT_REGISTERS, etc.) |

**Common SPL Queries:**
```spl
# Monitor register values over time
index=edge_hub_modbus deviceName="flow_meter"
| timechart avg(value) by startingAddress

# Detect communication failures
index=edge_hub_logs "modbus" ("Failed" OR "error" OR "timeout")
| stats count by hub_name, message
```

### MQTT

Subscribes to MQTT topics for IoT sensor data, including Meraki camera analytics.

**Index:** `edge_hub_mqtt`  
**Source:** `edgehub/mqtt_events/<topic_path>`  
**Sourcetypes:** `meraki_mt_json`, `meraki_mv_json`, `_json`

**Meraki MT Sensor Schema:**
```json
{
  "hub_name": "TV_prod_250dev_b",
  "ts": "2026-01-27T10:05:47Z",
  "rssi": -82,
  "units": "dBm"
}
```

**Meraki MV Camera Schema:**
```json
{
  "hub_name": "ContactTV-Kam-Hub",
  "ts": 1769509500,
  "zones": [{"id": "0", "person": 3}],
  "serial": "Q2HV-D2Y9-BHDC"
}
```

**Source Path Patterns:**
- `/merakimv/<serial>/0` - Zone-based people counting
- `/merakimv/<serial>/light` - Light level readings
- `/merakimv/<serial>/raw_detections` - Raw object detections
- `/merakimv/<serial>/audio_analytics` - Audio event detection
- `meraki/v1/mt/<network_id>/ble/<mac>/...` - Meraki MT sensor data

### Built-in Sensors

Edge Hub devices include onboard environmental sensors.

**Index:** `edge_hub_ot`  
**Sourcetype:** `edge_hub_ot`

**Source Patterns:**
| Source | Sensor Type | Fields |
|--------|-------------|--------|
| `edgehub/builtin/temperature/values` | Temperature | `edgehub.sensor.temperature` |
| `edgehub/builtin/humidity/values` | Humidity | `edgehub.sensor.humidity` |
| `edgehub/builtin/pressure/values` | Barometric | `edgehub.sensor.pressure` |
| `edgehub/builtin/light/values` | Ambient light | `edgehub.sensor.light` |
| `edgehub/builtin/sound/values` | Sound level | `edgehub.sensor.sound` |
| `edgehub/builtin/iaq/values` | Indoor air quality | `edgehub.sensor.iaq` |
| `edgehub/builtin/eCO2/values` | CO2 equivalent | `edgehub.sensor.eCO2` |
| `edgehub/builtin/gas/values` | Gas resistance | `edgehub.sensor.gas` |
| `edgehub/builtin/accelerometer/values` | Motion (3-axis) | `edgehub.sensor.accel_x/y/z` |
| `edgehub/builtin/gyroscope/values` | Rotation (3-axis) | `edgehub.sensor.gyro_x/y/z` |

**External Sensor Sources:**
| Source | Sensor Type |
|--------|-------------|
| `edgehub/external/temperature_external/values` | External temperature probe |
| `edgehub/external/humidity_external/values` | External humidity sensor |
| `edgehub/external/leak_external/values` | Water leak detection |

**Data Schema:**
```json
{
  "hub_name": "ContactTV-Kam-Hub",
  "edgehub.sensor.temperature": 23.45,
  "type": "temperature",
  "sensor_category": "builtin"
}
```

### OPC UA

Connects to OPC UA servers for industrial automation data.

**Index:** `edge_hub_opcua`  
**Source:** `edgehub/opcua`  
**Sourcetype:** `_json`

**Expected Schema:**
```json
{
  "hub_name": "factory-hub-01",
  "nodeId": "ns=2;s=Channel1.Device1.Tag1",
  "displayName": "Motor_Speed",
  "value": 1450.5,
  "statusCode": "Good",
  "serverTimestamp": "2026-01-27T10:00:00Z",
  "sourceTimestamp": "2026-01-27T10:00:00Z"
}
```

### SNMP

Polls network devices via SNMP for metrics and status.

**Index:** `edge_hub_snmp`  
**Source:** `edgehub/snmp`  
**Sourcetype:** `_json`

### Zeek Network Logs

Captures network traffic analysis from Zeek IDS.

**Index:** `edge_hub_zeek`  
**Source:** `edgehub/zeek/<log_type>`  
**Sourcetype:** `_json`

### Edge Hub System Logs

Operational logs from the Edge Hub device itself.

**Index:** `edge_hub_logs`  
**Source:** `edgehub/logs`  
**Sourcetype:** `_json`

**Log Schema:**
```json
{
  "hub_name": "TV_DVT2_240b",
  "unit": "init.scope",
  "message": "Stopped Splunk Edge Hub Modbus Client.",
  "boot_id": "4126cc9f704341df8b7d0cd54598ed7e",
  "priority": "6"
}
```

**Priority Levels:**
| Priority | Severity |
|----------|----------|
| 0-2 | Critical/Alert/Emergency |
| 3 | Error |
| 4 | Warning |
| 5 | Notice |
| 6 | Informational |
| 7 | Debug |

---

## OTI Datastreamer Ingest Methods

### HTTP Event Collector (HEC)

Primary ingest method for Edge Hub devices.

**Endpoint Format:**
```
https://<stack>.splunkcloud.com:8088/services/collector/event
```

**Headers:**
```
Authorization: Splunk <HEC_TOKEN>
Content-Type: application/json
```

**Payload Format:**
```json
{
  "event": { "field1": "value1" },
  "index": "edge_hub_ot",
  "sourcetype": "edge_hub_ot",
  "source": "edgehub/builtin/temperature/values",
  "host": "ACT-076-1823-0086",
  "time": 1769509500.123
}
```

**Batch Payload (multiple events):**
```json
{"event": {...}, "index": "edge_hub_ot", "time": 1769509500.0}
{"event": {...}, "index": "edge_hub_ot", "time": 1769509501.0}
{"event": {...}, "index": "edge_hub_ot", "time": 1769509502.0}
```

**HEC Troubleshooting:**
```spl
# Check HEC ingest health
index=_internal sourcetype=splunkd component=HttpInputDataHandler
| stats count by log_level, message

# Verify data arrival
index=edge_hub_* earliest=-5m
| stats count by index, sourcetype, host
```

### Splunk Connect for Kafka

For high-volume streaming from Kafka topics.

**Architecture:**
```
Edge Hub → Kafka Broker → Splunk Connect for Kafka → HEC → Splunk
```

**Connector Configuration:**
```properties
name=splunk-sink-edge-hub
connector.class=com.splunk.kafka.connect.SplunkSinkConnector
topics=edgehub.ot,edgehub.modbus
splunk.hec.uri=https://<stack>.splunkcloud.com:8088
splunk.hec.token=<HEC_TOKEN>
splunk.indexes=edge_hub_ot,edge_hub_modbus
```

### Heavy Forwarder Aggregation

For on-premises data aggregation before cloud ingest.

**outputs.conf:**
```ini
[httpout]
httpEventCollectorToken = <HEC_TOKEN>
uri = https://<stack>.splunkcloud.com:8088
```

---

## Index Reference

| Index | Purpose | Primary Sourcetypes |
|-------|---------|---------------------|
| `edge_hub_ot` | OT sensor data (builtin + general) | `edge_hub_ot` |
| `edge_hub_modbus` | Modbus TCP register data | `_json` |
| `edge_hub_mqtt` | MQTT subscriptions | `meraki_mt_json`, `meraki_mv_json` |
| `edge_hub_opcua` | OPC UA node values | `_json` |
| `edge_hub_snmp` | SNMP polling data | `_json` |
| `edge_hub_logs` | Edge Hub system logs | `_json` |
| `edge_hub_zeek` | Zeek network analysis | `_json` |
| `edge_hub_ws` | WebSocket streams | `_json` |
| `bms` | Building Management System | `bms_json` |

---

## Common SPL Patterns

### Multi-Hub Overview
```spl
index=edge_hub_* earliest=-1h
| stats count latest(_time) as last_seen by host, hub_name, index
| eval lag_seconds=now()-last_seen
| where lag_seconds > 300
| table hub_name, host, index, lag_seconds
```

### Sensor Value Aggregation
```spl
index=edge_hub_ot source="edgehub/builtin/*/values"
| rex field=source "edgehub/builtin/(?<sensor_type>[^/]+)/values"
| stats avg(*sensor*) as avg_value by sensor_type, hub_name
| table hub_name, sensor_type, avg_value
```

### Modbus Data Quality
```spl
index=edge_hub_modbus
| stats count dc(value) as unique_values by deviceName, startingAddress
| where unique_values < 2
| eval status="Stuck Value Alert"
```

### Camera People Counting
```spl
index=edge_hub_mqtt sourcetype=meraki_mv_json source="*merakimv*/0"
| spath zones{} output=zone_data
| mvexpand zone_data
| spath input=zone_data path=person output=person_count
| timechart span=5m sum(person_count) by source
```

---

## Troubleshooting

### No Data Arriving
1. Check Edge Hub connectivity: `index=edge_hub_logs hub_name="<name>" | head 10`
2. Verify HEC endpoint accessibility from Edge Hub network
3. Check for clock skew (events indexed with future timestamps)
4. Review Edge Hub service status in logs: `index=edge_hub_logs "Failed" OR "error"`

### Missing Sourcetypes
```spl
| metadata type=sourcetypes index=edge_hub_*
| table sourcetype, totalCount, recentTime
```

### Hub Health Dashboard Query
```spl
index=edge_hub_logs OR index=edge_hub_ot
| stats count(eval(index="edge_hub_logs")) as log_events
        count(eval(index="edge_hub_ot")) as sensor_events
        latest(_time) as last_seen
  by hub_name
| eval health=case(
    last_seen < relative_time(now(), "-5m"), "healthy",
    last_seen < relative_time(now(), "-15m"), "degraded",
    true(), "offline"
  )
```
