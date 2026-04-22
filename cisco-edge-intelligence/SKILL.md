---
name: cisco-edge-intelligence
description: "Cisco Edge Intelligence deployment and Splunk HEC integration guide. Use when: (1) Deploying Cisco Edge Intelligence on IR1101, IR829, IC3000, or Catalyst IE switches, (2) Configuring industrial data sources (OPC-UA, Modbus TCP/Serial, MQTT, Serial, NTCIP), (3) Creating data pipelines with Data Rules or Data Logic (JavaScript transforms), (4) Setting up Splunk HEC as a destination for operational telemetry, (5) Troubleshooting EI agent connectivity and data flow. Part of the Splunk VISTA for building operational data pipelines into Splunk."
---

# Cisco Edge Intelligence to Splunk Integration

## Overview

Cisco Edge Intelligence (EI) is IoT data orchestration software that extracts, transforms, governs, and delivers connected asset data from the IoT edge to cloud destinations. This skill covers deploying EI and integrating it with Splunk via HTTP Event Collector (HEC) as part of the Splunk VISTA methodology.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CISCO EDGE INTELLIGENCE ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  INDUSTRIAL ASSETS          EDGE DEVICE              SPLUNK PLATFORM        │
│  ─────────────────          ───────────              ───────────────         │
│                                                                              │
│  ┌─────────────┐     ┌────────────────────┐     ┌──────────────────┐        │
│  │ PLCs        │     │  Cisco IR1101/829  │     │                  │        │
│  │ (Modbus)    │────▶│  IC3000/Cat IE     │     │  Splunk Cloud    │        │
│  └─────────────┘     │                    │     │  or Enterprise   │        │
│                      │  ┌──────────────┐  │     │                  │        │
│  ┌─────────────┐     │  │ Edge Intel   │  │     │  ┌────────────┐  │        │
│  │ SCADA       │────▶│  │ Agent (IOx)  │──┼────▶│  │    HEC     │  │        │
│  │ (OPC-UA)    │     │  │              │  │     │  │  Endpoint  │  │        │
│  └─────────────┘     │  │ - Extract    │  │     │  └────────────┘  │        │
│                      │  │ - Transform  │  │     │        │         │        │
│  ┌─────────────┐     │  │ - Govern     │  │     │        ▼         │        │
│  │ IoT Sensors │────▶│  │ - Deliver    │  │     │  ┌────────────┐  │        │
│  │ (MQTT)      │     │  └──────────────┘  │     │  │  Indexes   │  │        │
│  └─────────────┘     └────────────────────┘     │  │ Dashboards │  │        │
│                                                  │  │  Alerts    │  │        │
│  ┌─────────────┐                                │  └────────────┘  │        │
│  │ Legacy      │                                └──────────────────┘        │
│  │ (Serial)    │─────────────────────────────────────────────────────       │
│  └─────────────┘                                                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Supported Deployment Platforms

| Device | Model | Description | Typical Use Case |
|--------|-------|-------------|------------------|
| Industrial Router | Cisco IR1101 | Compact industrial router with IOx | Remote sites, pump stations, substations |
| Industrial Router | Cisco IR829 | Rugged industrial router | Harsh environments, mobile/vehicle |
| Compute Gateway | Cisco IC3000 | Industrial compute gateway | Heavy edge processing needs |
| Industrial Switch | Cisco Catalyst IE | Industrial Ethernet with IOx | Plant floor with compute needs |

**Licensing**: Edge Intelligence requires Network Advantage licenses on network devices.

---

## Supported Industrial Protocols (Data Sources)

| Protocol | Connection Type | Use Case | Key Configuration |
|----------|-----------------|----------|-------------------|
| **OPC-UA** | TCP/IP | PLCs, SCADA, historians | IP, port, namespace, node IDs |
| **Modbus TCP** | TCP/IP | Industrial controllers, PLCs | IP, port, slave ID, registers |
| **Modbus RTU** | Serial (RS-485/232) | Legacy equipment | Serial port, baud rate, parity |
| **MQTT** | TCP/IP | IoT sensors, gateways | Broker URL, topics, TLS |
| **Serial** | RS-232/485 | Legacy devices, sensors | Port, baud, start/end codes |
| **NTCIP** | TCP/IP | Traffic systems (1202/1203/1204) | IP, SNMP version, OIDs |
| **RSU** | TCP/IP | Roadside units (V2X) | IP, port, SNMP settings |

---

## Pipeline Architecture

Edge Intelligence uses a **pipeline-based architecture** for data flow:

```
┌─────────────────────────────────────────────────────────────────┐
│                        PIPELINE STRUCTURE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │   SOURCE    │    │   POLICY    │    │    DESTINATION      │  │
│  │             │    │             │    │                     │  │
│  │ • OPC-UA    │    │ • Data Rule │    │ • Splunk HEC        │  │
│  │ • Modbus    │───▶│   (pass-    │───▶│ • MQTT Server       │  │
│  │ • MQTT      │    │   through)  │    │ • Azure IoT Hub     │  │
│  │ • Serial    │    │             │    │ • AWS IoT Core      │  │
│  │ • NTCIP     │    │ • Data Logic│    │                     │  │
│  │             │    │   (JS xform)│    │                     │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
│                                                                  │
│  Up to 20 sources        Transform or          One destination   │
│  per pipeline            pass-through          per pipeline      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Pipeline Naming Conventions
- Use **CamelCase** (e.g., `WaterSensorSalinityJ2345`)
- Avoid special characters (removed during processing)
- Do not end with `s` (causes naming conflicts)
- Names must be unique per agent

---

## Splunk HEC Destination Configuration

Splunk is supported as a native destination starting in **Release 2.2.x**.

### Prerequisites
1. Active Splunk instance (Cloud or Enterprise)
2. HTTP Event Collector (HEC) enabled
3. HEC token created with appropriate index permissions
4. Network connectivity from edge device to Splunk HEC endpoint

### Splunk Destination Settings

| Field | Description | Example |
|-------|-------------|---------|
| **HEC URL** | Full URL to HEC endpoint | `https://splunk.company.com:8088/services/collector/event` |
| **HEC Token** | Authentication token | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| **Enable TLS** | Required for HTTPS | ✓ Enabled |
| **Verify Server Certificate** | Validate indexer cert | Optional - upload CA bundle |
| **Enable Mutual TLS (mTLS)** | Client certificate auth | Optional - upload client cert/key |

### HEC URL Format
```
https://<splunk-host>:8088/services/collector/event
```

For Splunk Cloud:
```
https://<stack>.splunkcloud.com:8088/services/collector/event
```

---

## Data Policies

### Data Rule (Pass-through)
Send raw data directly from sources to Splunk without transformation.

**Limitations**: 
- NOT supported for Splunk destination
- Use Data Logic for Splunk

### Data Logic (JavaScript Transform)
Transform data before sending to Splunk. **Required for Splunk destination**.

**Engine**: Duktape 2.7.0 (ES5.0/ES5.1 compliant)

#### Required Functions

```javascript
// Called once when pipeline starts
function init() {
    logger.info("Pipeline initialized");
    // Configure HTTP/SSL options here
}

// Called on each data update from source
function on_update() {
    // Access source data via input object
    var temperature = input.temperature;
    // Transform and publish
}

// Called at configured interval (optional)
function on_time_trigger() {
    // Periodic processing
}
```

#### Global Objects

| Object | Description |
|--------|-------------|
| `input` | Source data fields (read-only) |
| `output` | Destination data fields (set values) |
| `trigger` | Trigger metadata (`field_name`, `device_name`, `timestamp`) |
| `parameters` | Runtime configuration parameters |
| `logger` | Logging interface (`info`, `warn`, `error`, `debug`) |

---

## Splunk Data Logic Examples

### Single Payload Example

```javascript
function init() {
    logger.info("Starting Splunk integration");
}

var counter = 0;

function on_update() {
    // Not used for time-triggered sends
}

function on_time_trigger() {
    counter = counter + 1;
    
    // Create Splunk HEC event payload
    var payload = {
        event: {
            temperature: input.temperature,
            humidity: input.humidity,
            pressure: input.pressure,
            counter: counter
        },
        host: input.asset_serial_number,
        source: "cisco:ei:telemetry",
        sourcetype: "cisco:ei:telemetry"
        // Optional: "index": "cisco_ei"
        // Optional: "time": new Date(trigger.timestamp).getTime() / 1000
    };
    
    // Publish to Splunk
    publish("output", payload);
}
```

### Batch Payload Example

```javascript
function init() {
    logger.info("Batch mode initialized");
}

var messageBuffer = [];
var maxBufferSize = 10;

function on_update() {
    var payload = {
        event: {
            metric: trigger.field_name,
            value: input[trigger.field_name],
            asset: input.asset_serial_number,
            timestamp: trigger.timestamp
        },
        host: input.asset_serial_number,
        source: "cisco:ei:modbus",
        sourcetype: "cisco:ei:modbus"
    };
    
    messageBuffer.push(payload);
    
    // Send batch when buffer is full
    if (messageBuffer.length >= maxBufferSize) {
        publish("output", messageBuffer);
        messageBuffer = [];
    }
}

function on_time_trigger() {
    // Flush remaining buffer periodically
    if (messageBuffer.length > 0) {
        publish("output", messageBuffer);
        messageBuffer = [];
    }
}
```

### Threshold Alert Example

```javascript
var thresholds = {
    temperature_high: 85,
    pressure_low: 10,
    vibration_high: 100
};

function on_update() {
    var field = trigger.field_name;
    var value = input[field];
    var alert = null;
    
    // Check thresholds
    if (field === "temperature" && value > thresholds.temperature_high) {
        alert = "HIGH_TEMPERATURE";
    } else if (field === "pressure" && value < thresholds.pressure_low) {
        alert = "LOW_PRESSURE";
    } else if (field === "vibration" && value > thresholds.vibration_high) {
        alert = "HIGH_VIBRATION";
    }
    
    if (alert) {
        var payload = {
            event: {
                alert_type: alert,
                metric: field,
                value: value,
                threshold: thresholds[field + "_high"] || thresholds[field + "_low"],
                asset_id: input.asset_serial_number,
                severity: "critical"
            },
            host: input.asset_serial_number,
            source: "cisco:ei:alert",
            sourcetype: "cisco:ei:alert"
        };
        
        publish("output", payload);
        logger.warn("Alert triggered: " + alert + " = " + value);
    }
}
```

### Sliding Average Example

```javascript
function SlidingWindow(size) {
    this.size = size;
    this.values = [];
    this.sum = 0;
}

SlidingWindow.prototype.update = function(value) {
    if (this.values.length >= this.size) {
        this.sum -= this.values.shift();
    }
    this.sum += value;
    this.values.push(value);
};

SlidingWindow.prototype.average = function() {
    return this.values.length > 0 ? this.sum / this.values.length : 0;
};

var windows = {};
var windowSize = 10;

function init() {
    windows.temperature = new SlidingWindow(windowSize);
    windows.pressure = new SlidingWindow(windowSize);
    logger.info("Sliding average initialized with window size: " + windowSize);
}

function on_update() {
    var field = trigger.field_name;
    var value = input[field];
    
    if (windows[field]) {
        windows[field].update(value);
        
        var payload = {
            event: {
                metric: field,
                raw_value: value,
                avg_value: windows[field].average(),
                sample_count: windows[field].values.length,
                asset_id: input.asset_serial_number
            },
            host: input.asset_serial_number,
            source: "cisco:ei:analytics",
            sourcetype: "cisco:ei:analytics"
        };
        
        publish("output", payload);
    }
}
```

---

## Source Configuration Examples

### OPC-UA Data Model

```json
{
  "apiVersion": 1,
  "connectionType": "OPC_UA",
  "fields": {
    "temperature": {
      "label": "Temperature",
      "description": "Process temperature sensor",
      "datatype": "Float",
      "nodeId": {
        "namespaceUri": "2",
        "identifier": "Channel1.Device1.Temperature",
        "type": "string"
      },
      "samplingInterval": 1000,
      "category": "TELEMETRY"
    },
    "pressure": {
      "label": "Pressure",
      "description": "Process pressure sensor",
      "datatype": "Float",
      "nodeId": {
        "namespaceUri": "2",
        "identifier": "Channel1.Device1.Pressure",
        "type": "string"
      },
      "samplingInterval": 1000,
      "category": "TELEMETRY"
    }
  }
}
```

### Modbus TCP Data Model

```json
{
  "apiVersion": 1,
  "connectionType": "MODBUS_TCP",
  "fields": {
    "flow_rate": {
      "label": "Flow Rate",
      "datatype": "Float",
      "description": "Inlet flow rate",
      "rawType": "FLOAT32",
      "type": "HOLDING",
      "pollingInterval": 5000,
      "offset": 100,
      "category": "TELEMETRY",
      "access": "Read"
    },
    "tank_level": {
      "label": "Tank Level",
      "datatype": "Int",
      "description": "Tank level percentage",
      "rawType": "UINT16",
      "type": "INPUT",
      "pollingInterval": 5000,
      "offset": 200,
      "category": "TELEMETRY",
      "access": "Read"
    },
    "setpoint": {
      "label": "Temperature Setpoint",
      "datatype": "Int",
      "description": "Target temperature",
      "rawType": "UINT16",
      "type": "HOLDING",
      "pollingInterval": 10000,
      "offset": 50,
      "category": "TELEMETRY",
      "access": "ReadWrite"
    }
  }
}
```

### MQTT Data Model

```json
{
  "apiVersion": 1.0,
  "connectionType": "MQTT",
  "fields": {
    "temperature": {
      "category": "TELEMETRY",
      "label": "Temperature",
      "description": "Environmental sensor temperature",
      "datatype": "Float",
      "topic": "sensors/env001/temperature"
    },
    "humidity": {
      "category": "TELEMETRY",
      "label": "Humidity",
      "description": "Environmental sensor humidity",
      "datatype": "Float",
      "topic": "sensors/env001/humidity"
    },
    "co2": {
      "category": "TELEMETRY",
      "label": "CO2 Level",
      "description": "CO2 concentration in ppm",
      "datatype": "Int",
      "topic": "sensors/env001/co2"
    }
  }
}
```

---

## Splunk Configuration

### Create HEC Token

1. In Splunk Web: **Settings → Data Inputs → HTTP Event Collector**
2. Click **New Token**
3. Configure:
   - **Name**: `cisco_edge_intelligence`
   - **Source type**: `cisco:ei:telemetry` (or allow override)
   - **Index**: `cisco_ei` (create if needed)
   - **Enable indexer acknowledgment**: Optional (reduces throughput)
4. Copy the token value for EI configuration

### Recommended Indexes

| Index | Sourcetype | Purpose |
|-------|------------|---------|
| `cisco_ei` | `cisco:ei:telemetry` | General telemetry data |
| `cisco_ei` | `cisco:ei:modbus` | Modbus register data |
| `cisco_ei` | `cisco:ei:opcua` | OPC-UA node values |
| `cisco_ei` | `cisco:ei:mqtt` | MQTT message payloads |
| `cisco_ei` | `cisco:ei:alert` | Threshold alerts |
| `cisco_ei` | `cisco:ei:analytics` | Processed/aggregated data |

### Recommended Sourcetypes

Create `props.conf` entries:

```ini
[cisco:ei:telemetry]
TIME_FORMAT = %s.%3N
TIME_PREFIX = "timestamp"\s*:\s*
MAX_TIMESTAMP_LOOKAHEAD = 20
SHOULD_LINEMERGE = false
KV_MODE = json
TRUNCATE = 50000

[cisco:ei:modbus]
TIME_FORMAT = %s.%3N
TIME_PREFIX = "timestamp"\s*:\s*
SHOULD_LINEMERGE = false
KV_MODE = json

[cisco:ei:opcua]
TIME_FORMAT = %s.%3N
TIME_PREFIX = "timestamp"\s*:\s*
SHOULD_LINEMERGE = false
KV_MODE = json

[cisco:ei:alert]
TIME_FORMAT = %s.%3N
TIME_PREFIX = "timestamp"\s*:\s*
SHOULD_LINEMERGE = false
KV_MODE = json
```

---

## Sample SPL Queries

### Equipment Telemetry Overview

```spl
index=cisco_ei sourcetype="cisco:ei:telemetry"
| stats latest(value) as current_value 
        avg(value) as avg_value 
        min(value) as min_value 
        max(value) as max_value 
  by host, metric
| table host, metric, current_value, avg_value, min_value, max_value
```

### Modbus Register Trends

```spl
index=cisco_ei sourcetype="cisco:ei:modbus"
| timechart span=5m avg(value) by metric
```

### Alert Analysis

```spl
index=cisco_ei sourcetype="cisco:ei:alert"
| stats count by alert_type, asset_id, severity
| sort - count
```

### Data Throughput Monitoring

```spl
index=cisco_ei
| timechart span=1h count by sourcetype
| addtotals
```

### Edge Device Health

```spl
index=cisco_ei 
| stats latest(_time) as last_seen 
        count as event_count 
  by host
| eval lag_seconds = now() - last_seen
| eval status = case(
    lag_seconds < 300, "healthy",
    lag_seconds < 900, "degraded",
    true(), "offline"
  )
| table host, status, lag_seconds, event_count
```

### Anomaly Detection

```spl
index=cisco_ei sourcetype="cisco:ei:telemetry" metric="temperature"
| streamstats window=20 avg(value) as moving_avg stdev(value) as moving_stdev
| eval upper_bound = moving_avg + (2 * moving_stdev)
| eval lower_bound = moving_avg - (2 * moving_stdev)
| eval anomaly = if(value > upper_bound OR value < lower_bound, 1, 0)
| where anomaly = 1
| table _time, host, metric, value, moving_avg, upper_bound, lower_bound
```

---

## Troubleshooting

### Common Issues

| Symptom | Possible Cause | Resolution |
|---------|----------------|------------|
| No data in Splunk | HEC token invalid | Verify token and permissions |
| No data in Splunk | Network blocked | Check firewall rules for port 8088 |
| No data in Splunk | TLS certificate error | Upload CA cert or disable verification |
| Pipeline won't deploy | Invalid data model | Check JSON syntax and field definitions |
| Source shows offline | Network unreachable | Verify IP/port of source device |
| Source shows offline | Wrong credentials | Check Modbus slave ID, OPC-UA auth |
| Partial data | Polling interval too fast | Increase polling interval |
| Duplicate events | Multiple pipelines | Consolidate sources into single pipeline |

### Health Status Check

In EI Local Manager:
1. Click pipeline name
2. Select **Health Status** tab
3. Review:
   - **Pipeline Status**: Overall health
   - **Source Status**: Connection to data sources
   - **Destination Status**: Connection to Splunk HEC

### Logging Levels

In Data Logic scripts:
```javascript
logger.trace("Detailed tracing");
logger.debug("Debug information");
logger.info("Informational messages");
logger.warn("Warning conditions");
logger.error("Error conditions");
logger.fatal("Critical failures");
```

**Note**: Set `productive = false` in pipeline to enable log emission.

### HEC Connectivity Test

```bash
# From edge device or network with access
curl -k "https://<splunk-host>:8088/services/collector/event" \
  -H "Authorization: Splunk <HEC_TOKEN>" \
  -d '{"event": "test", "sourcetype": "cisco:ei:test"}'
```

Expected response:
```json
{"text":"Success","code":0}
```

---

## Best Practices

### Edge Filtering
Reduce bandwidth by filtering at the edge:
- **Send only on change**: Avoid sending static values
- **Use deadbands**: Only send when value changes by threshold
- **Aggregate high-frequency data**: Use sliding averages

### Batch vs Single Payload
| Mode | Use When | Benefits |
|------|----------|----------|
| **Batch** | High-volume telemetry | Reduces HTTP overhead |
| **Single** | Critical alerts | Lower latency |

### Data Governance
- Use **data policies** to control routing
- Implement **role-based access** to pipelines
- **Version control** pipeline configurations (export as JSON templates)

### Splunk Optimization
- Use **indexed fields** for common search terms (`asset_id`, `metric`)
- Set appropriate **retention policies** based on data value
- Consider **summary indexing** for long-term trends

---

## Integration with Splunk VISTA

This skill supports the **Splunk VISTA methodology**:

### Phase 1: Research
- Identify industrial data sources (PLCs, sensors, SCADA)
- Map protocols to EI capabilities (OPC-UA, Modbus, MQTT)
- Document network topology and connectivity requirements

### Phase 2: Design
- Design EI pipeline architecture
- Define data models for each source
- Plan Data Logic transformations
- Specify Splunk index and sourcetype strategy

### Phase 3: Build
- Deploy EI agent on Cisco devices
- Configure source connections
- Implement Data Logic scripts
- Set up Splunk HEC destination

### Phase 4: Validate
- Verify data flow from sources to Splunk
- Test Data Logic transformations
- Validate field extractions and parsing
- Confirm dashboard and alert functionality

### Phase 5: Document
- Export pipeline templates (JSON)
- Document data models and field mappings
- Create runbooks for operations
- Prepare user training materials

---

## Related Skills

- **splunk-edge-hub**: For Splunk Edge Hub (different product) integration
- **splunk-admin**: For Splunk configuration, SPL, and knowledge objects
- **cisco-splunk-integration**: For other Cisco products (Meraki, ISE, etc.)

---

## References

- [Cisco Edge Intelligence User Guide](https://www.cisco.com/c/en/us/td/docs/iot/Cisco-Edge-Intelligence/b-cisco-edge-intelligence-user-guide-2-2-x.html)
- [Cisco Edge Intelligence API Documentation](https://developer.cisco.com/docs/edge-intelligence/)
- [Splunk HTTP Event Collector Guide](https://docs.splunk.com/Documentation/Splunk/latest/Data/UsetheHTTPEventCollector)
