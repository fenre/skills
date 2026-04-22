---
name: splunk-oti-data-model
description: >
  Operational Telemetry (OT/IoT) data model configuration skill. Use this skill when:
  (1) Normalizing industrial sensor data into the Operational_Telemetry data model — Metrics, Events,
  States, Production, OEE, Quality, Maintenance, Security, Location objects;
  (2) Writing props.conf field aliases and calculated fields for OT protocol data (Modbus, OPC-UA,
  BACnet, MQTT, Meraki); (3) Configuring eventtypes.conf and tags.conf for OT data model routing;
  (4) Mapping metric_name, metric_value, and metric_unit fields for universal sensor readings;
  (5) Validating OT data model compliance with tstats queries;
  (6) Understanding CIM vs Operational Telemetry data model selection for OT/IoT use cases.
---

# Operational Telemetry Data Model - Splunk Configuration Skill

## Overview

The **Operational_Telemetry** data model provides a normalized schema for Industrial IoT, Building Management, Smart City, and Event/Venue telemetry data. This skill provides expert guidance on normalizing source data into the data model using Splunk knowledge objects.

**Key Principle**: All numeric sensor readings use the universal **Metrics** object with `metric_name`, `metric_value`, and `metric_unit` fields rather than dedicated fields per metric type.

### Supported Verticals

| Vertical | Use Cases |
|----------|-----------|
| **Manufacturing** | OEE, production counts, quality SPC, equipment monitoring |
| **Facilities/BMS** | HVAC, lighting, energy management, occupancy |
| **Smart Cities** | Traffic, parking, air quality, utilities, street lighting |
| **Events/Venues** | Crowd flow, ticketing, concessions, parking, security |
| **Critical Infrastructure** | Water/wastewater, power grid, oil & gas |

### Supported Protocols

| Protocol | Typical Sources |
|----------|-----------------|
| **Modbus TCP/RTU** | PLCs, VFDs, meters, industrial sensors |
| **OPC-UA** | Modern PLCs, SCADA, historians |
| **BACnet** | Building automation, HVAC controllers |
| **MQTT** | IoT sensors, edge gateways |
| **DNP3** | Utilities, SCADA systems |
| **REST/API** | Cloud platforms (Meraki, Spaces, etc.) |

---

## Quick-Start Guide

### 5-Minute Setup for a New Data Source

**Step 1**: Identify your sourcetype and index
```spl
index=* earliest=-1h | stats count by index, sourcetype | sort -count
```

**Step 2**: Create minimal props.conf entry
```ini
[your_sourcetype]
KV_MODE = json
SHOULD_LINEMERGE = false

# Map the 5 required base fields
EVAL-device_id = 'your_device_field'
EVAL-device_type = "sensor"
EVAL-vertical = "facilities"
EVAL-site = "your_site"
EVAL-system = "your_system"

# Map to universal metric pattern
EVAL-metric_name = "your_metric"
EVAL-metric_value = tonumber('your_value_field')
EVAL-metric_unit = "your_unit"
```

**Step 3**: Verify in Splunk
```spl
index=your_index sourcetype=your_sourcetype earliest=-15m
| table _time device_id device_type vertical site system metric_name metric_value metric_unit
```

**Step 4**: Verify data model mapping
```spl
| tstats count FROM datamodel=Operational_Telemetry WHERE nodename=All_Telemetry.Metrics index=your_index BY All_Telemetry.Metrics.metric_name
```

---

## Data Model Architecture

### Object Hierarchy

```
Operational_Telemetry
├── All_Telemetry (base)          # Common device/location fields
│   ├── Metrics                   # ALL numeric measurements
│   ├── Events                    # Alarms, faults, notifications
│   ├── States                    # Equipment modes/positions
│   ├── Production                # Manufacturing counts/batches
│   ├── OEE                       # Overall Equipment Effectiveness
│   ├── Quality                   # SPC measurements
│   ├── Maintenance               # Work orders, predictive scores
│   ├── Security                  # Access control, intrusion
│   ├── VenueOperations           # Events/venues scheduling
│   └── Location                  # Real-time positioning
```

### Required Fields by Object

| Object | Required Fields | Constraint |
|--------|-----------------|------------|
| **All_Telemetry** | `device_id`, `device_type`, `vertical`, `site`, `system` | Base search constraint |
| **Metrics** | `metric_name`, `metric_value` | `metric_name=* metric_value=*` |
| **Events** | `event_type` | `event_type=*` |
| **States** | `state_name`, `state_value` | `state_name=* state_value=*` |
| **Production** | At least one of: `good_count`, `batch_id`, `production_order` | |
| **OEE** | At least one of: `oee`, `availability`, `downtime_minutes` | |
| **Quality** | `measurement_name`, `measurement_value` | |
| **Maintenance** | At least one of: `work_order_id`, `health_score`, `failure_probability` | |
| **Security** | At least one of: `access_point`, `intrusion_zone`, `camera_id`, `credential_id`, `ticket_id` | |
| **VenueOperations** | At least one of: `event_id`, `venue_id`, `gate_id`, `route_id` | |
| **Location** | At least one of: `location_id`, `tracked_device_id`, `visit_id` | |

---

## Standard Vocabularies

### metric_name Values

Use these standardized names for consistency:

**Environmental**
```
temperature, humidity, dewpoint, co2, voc, pm25, pm10, noise_level, 
light_level, air_quality_index, pressure, wind_speed, wind_direction
```

**Electrical**
```
power, apparent_power, reactive_power, voltage, current, frequency, 
power_factor, energy, energy_delta, thd
```

**HVAC/Process**
```
supply_temp, return_temp, discharge_temp, zone_temp, outdoor_temp,
setpoint, flow_rate, static_pressure, differential_pressure, 
valve_position, damper_position, fan_speed, vfd_speed
```

**Utility (Water/Gas)**
```
flow_rate, pressure, level, ph, turbidity, chlorine_residual, 
dissolved_oxygen, conductivity, gas_consumption
```

**Performance/Counts**
```
person_count, vehicle_count, device_count, occupancy_count, 
queue_length, wait_time, throughput, dwell_time, cycle_time
```

**Device Health**
```
battery_level, signal_strength, uptime, cpu_usage, memory_usage
```

### metric_category Values

```
environmental, electrical, process, mechanical, performance, 
utility, hvac, network, device_health, occupancy
```

### metric_unit Values

| Category | Units |
|----------|-------|
| Temperature | `degC`, `degF`, `K` |
| Humidity | `%RH`, `percent` |
| Pressure | `Pa`, `kPa`, `bar`, `psi`, `inH2O`, `hPa` |
| Flow | `m3/h`, `L/min`, `gpm`, `cfm` |
| Power | `W`, `kW`, `MW`, `VA`, `kVA` |
| Energy | `Wh`, `kWh`, `MWh`, `J`, `BTU` |
| Voltage | `V`, `kV`, `mV` |
| Current | `A`, `mA` |
| Frequency | `Hz` |
| Air Quality | `ppm`, `ppb`, `µg/m³`, `mg/m³` |
| Light | `lux`, `lm`, `cd` |
| Sound | `dB`, `dBA` |
| Speed | `m/s`, `km/h`, `mph`, `rpm` |
| Count | `count`, `units`, `pieces` |
| Percentage | `percent`, `%` |

### state_name Values

```
run_status, mode, position, availability, door_state, valve_state,
damper_state, intrusion_state, presence, occupancy, alarm_state,
operating_mode, control_mode, schedule_mode
```

### state_value Values

| state_name | Allowed Values |
|------------|----------------|
| run_status | `running`, `stopped`, `starting`, `stopping`, `fault` |
| mode | `auto`, `manual`, `off`, `override`, `standby` |
| position | `open`, `closed`, `opening`, `closing`, `partial` |
| door_state | `open`, `closed`, `locked`, `unlocked`, `forced` |
| presence | `entry`, `exit`, `active`, `inactive` |
| occupancy | `occupied`, `unoccupied`, `standby` |

---

## Configuring Data Sources

### Step 1: Identify Index Constraints

The data model searches these indexes by default:
```
index=industrial OR index=iot OR index=manufacturing OR index=bms OR 
index=scada OR index=utilities OR index=infrastructure OR index=events_venue OR 
index=smart_city OR index=edge_hub_mqtt OR index=edge_hub_ot OR 
index=edge_hub_modbus OR index=edge_hub_opcua OR index=cisco_meraki OR index=cisco_spaces
```

To add a new index, update `datamodels.conf`:
```ini
# In datamodels.conf, find the acceleration stanza and update constraints
# or modify the JSON model constraints array
```

### Step 2: Create props.conf for Your Sourcetype

#### Template: Basic JSON Sourcetype

```ini
[your_sourcetype]
# Timestamp parsing
TIME_FORMAT = %Y-%m-%dT%H:%M:%S.%3NZ
MAX_TIMESTAMP_LOOKAHEAD = 35
SHOULD_LINEMERGE = false
LINE_BREAKER = ([\r\n]+)
TRUNCATE = 10000

# Enable JSON extraction
KV_MODE = json

#############################################################################
# FIELD EXTRACTIONS - Extract raw fields from your data
#############################################################################

# Option 1: EXTRACT for regex-based extraction
EXTRACT-device = "device_id"\s*:\s*"(?<device_id>[^"]+)"

# Option 2: EVAL for computed fields (preferred for JSON)
EVAL-device_id = 'sensor.id'
EVAL-device_name = 'sensor.name'

#############################################################################
# FIELD ALIASES - Map source fields to data model fields
#############################################################################

FIELDALIAS-device = sensor_id AS device_id
FIELDALIAS-value = reading AS metric_value

#############################################################################
# LOOKUPS - Enrich with external data
#############################################################################

LOOKUP-device_info = your_device_lookup device_id OUTPUT device_name site building floor zone system

#############################################################################
# CALCULATED FIELDS - Compute normalized fields
#############################################################################

# CRITICAL: Map to metric_name, metric_value, metric_unit pattern
EVAL-metric_name = case(\
    isnotnull('temperature'), "temperature",\
    isnotnull('humidity'), "humidity",\
    isnotnull('power'), "power",\
    1=1, "unknown")

EVAL-metric_value = coalesce(\
    tonumber('temperature'),\
    tonumber('humidity'),\
    tonumber('power'))

EVAL-metric_unit = case(\
    metric_name=="temperature", "degC",\
    metric_name=="humidity", "%RH",\
    metric_name=="power", "kW",\
    1=1, "")

# Set category for filtering
EVAL-metric_category = case(\
    metric_name IN ("temperature", "humidity", "co2"), "environmental",\
    metric_name IN ("power", "voltage", "current"), "electrical",\
    1=1, "other")

# Set required base fields
EVAL-device_type = "sensor"
EVAL-vertical = "facilities"
EVAL-system = "HVAC"
EVAL-protocol = "mqtt"
EVAL-metric_quality = "good"
```

### Step 3: Create Lookup Tables

#### Device Enrichment Lookup

Create `your_device_lookup.csv`:
```csv
device_id,device_name,device_type,site,site_name,building,floor,zone,system,subsystem,asset_id,criticality
SENSOR-001,AHU-1 Supply Temp,temperature_sensor,SITE_A,Main Campus,BLDG_1,2,Zone_A,HVAC,AHU-1,ASSET-001,high
SENSOR-002,Chiller 1 Power,power_meter,SITE_A,Main Campus,BLDG_1,B1,Mechanical,Electrical,Chiller_Plant,ASSET-002,critical
```

#### Metric Threshold Lookup

Create `metric_thresholds.csv`:
```csv
metric_name,metric_category,metric_unit,low_critical,low_warning,high_warning,high_critical,description
temperature,environmental,degC,10,15,28,35,Zone temperature
humidity,environmental,%RH,20,30,60,70,Relative humidity
power,electrical,kW,0,0,80,100,Equipment power consumption
co2,environmental,ppm,0,0,800,1000,CO2 concentration
```

Register lookups in `transforms.conf`:
```ini
[your_device_lookup]
filename = your_device_lookup.csv
case_sensitive_match = false

[metric_thresholds]
filename = metric_thresholds.csv
case_sensitive_match = false
```

Enable automatic lookup in `props.conf`:
```ini
[your_sourcetype]
LOOKUP-device_info = your_device_lookup device_id OUTPUT device_name site building floor zone system subsystem asset_id criticality
LOOKUP-thresholds = metric_thresholds metric_name OUTPUT metric_min metric_max low_critical low_warning high_warning high_critical
```

---

## Protocol-Specific Configuration Examples

### Modbus TCP/RTU

```ini
[modbus]
TIME_FORMAT = %Y-%m-%dT%H:%M:%S.%3N%z
SHOULD_LINEMERGE = false
LINE_BREAKER = ([\r\n]+)

# Extract Modbus-specific fields
EXTRACT-modbus_register = register=(?<register_address>\d+)\s+value=(?<raw_value>[\d.-]+)
EXTRACT-modbus_slave = slave_id=(?<modbus_slave_id>\d+)
EXTRACT-modbus_function = function_code=(?<modbus_function>\d+)

# Create device_id
EVAL-device_id = "modbus:".modbus_slave_id.":".register_address

# Use lookup to map register addresses to metric names
LOOKUP-registers = modbus_register_map register_address modbus_slave_id OUTPUT metric_name metric_unit device_name device_type

# Map Modbus function codes to quality
EVAL-metric_quality = case(\
    modbus_function IN ("1","2","3","4"), "good",\
    modbus_function IN ("129","130","131","132"), "bad",\
    1=1, "unknown")

EVAL-protocol = "modbus"
FIELDALIAS-value = raw_value AS metric_value
```

**modbus_register_map.csv**:
```csv
modbus_slave_id,register_address,metric_name,metric_unit,device_name,device_type,system
1,40001,temperature,degC,Boiler 1 Supply Temp,temperature_sensor,HVAC
1,40002,pressure,psi,Boiler 1 Pressure,pressure_sensor,HVAC
1,40003,flow_rate,gpm,Boiler 1 Flow,flow_meter,HVAC
2,40001,power,kW,Motor 1 Power,power_meter,Electrical
```

### OPC-UA

```ini
[opcua]
TIME_FORMAT = %Y-%m-%dT%H:%M:%S.%6NZ
SHOULD_LINEMERGE = false
LINE_BREAKER = ([\r\n]+)
KV_MODE = json

# Extract OPC-UA node information
EXTRACT-nodeid = "NodeId"\s*:\s*"(?<opcua_node_id>[^"]+)"
EXTRACT-value = "Value"\s*:\s*(?<raw_value>[\d.-]+|"[^"]*"|true|false)
EXTRACT-quality = "StatusCode"\s*:\s*"?(?<opcua_status>\w+)"?

# Create device_id from node ID
EVAL-device_id = "opcua:".replace(opcua_node_id, "ns=\d+;", "")
EVAL-tag_path = opcua_node_id

# Lookup for tag mapping
LOOKUP-tags = opcua_tag_map opcua_node_id OUTPUT metric_name metric_unit device_name device_type site system

# Map OPC-UA quality
EVAL-metric_quality = case(\
    opcua_status=="Good" OR opcua_status=="0", "good",\
    match(opcua_status, "Uncertain"), "suspect",\
    match(opcua_status, "Bad"), "bad",\
    1=1, "unknown")

EVAL-protocol = "opcua"
FIELDALIAS-value = raw_value AS metric_value
```

### BACnet

```ini
[bacnet]
TIME_FORMAT = %Y-%m-%dT%H:%M:%S%z
SHOULD_LINEMERGE = false
LINE_BREAKER = ([\r\n]+)

# BACnet object addressing
EXTRACT-device = device:(?<bacnet_device_id>\d+)
EXTRACT-object = object:(?<bacnet_object_type>\w+),(?<bacnet_instance>\d+)
EXTRACT-property = property:(?<bacnet_property>\w+)=(?<raw_value>[^\s,]+)
EXTRACT-units = units:(?<bacnet_units>\d+)

# Create composite device_id
EVAL-device_id = "bacnet:".bacnet_device_id.":".bacnet_object_type.":".bacnet_instance

# Lookup for object mapping
LOOKUP-objects = bacnet_object_map bacnet_object_type bacnet_instance OUTPUT metric_name device_name device_type site system subsystem

# Map BACnet engineering units (per ASHRAE standard)
LOOKUP-units = bacnet_unit_map bacnet_units OUTPUT metric_unit

EVAL-protocol = "bacnet"
EVAL-system = "BMS"
FIELDALIAS-value = raw_value AS metric_value
```

**bacnet_unit_map.csv** (partial - per ASHRAE 135):
```csv
bacnet_units,metric_unit
62,degC
64,degF
95,%RH
98,Pa
19,psi
118,kW
121,kWh
31,Hz
122,V
3,A
```

### MQTT JSON (Generic IoT)

```ini
[mqtt:sensor]
TIME_FORMAT = %Y-%m-%dT%H:%M:%S.%3NZ
SHOULD_LINEMERGE = false
LINE_BREAKER = ([\r\n]+)
KV_MODE = json

# Extract topic components from source
EXTRACT-topic = topic/(?<site>[^/]+)/(?<device_id>[^/]+)/(?<metric_name>[^/]+)$ in source

# Multi-metric JSON handling
EVAL-metric_name = case(\
    isnotnull('readings.temperature'), "temperature",\
    isnotnull('readings.humidity'), "humidity",\
    isnotnull('readings.pressure'), "pressure",\
    isnotnull('readings.co2'), "co2",\
    isnotnull('power.watts'), "power",\
    isnotnull('power.voltage'), "voltage",\
    isnotnull('power.current'), "current",\
    1=1, "unknown")

EVAL-metric_value = coalesce(\
    tonumber('readings.temperature'),\
    tonumber('readings.humidity'),\
    tonumber('readings.pressure'),\
    tonumber('readings.co2'),\
    tonumber('power.watts'),\
    tonumber('power.voltage'),\
    tonumber('power.current'))

EVAL-metric_unit = case(\
    metric_name=="temperature", "degC",\
    metric_name=="humidity", "%RH",\
    metric_name=="pressure", "hPa",\
    metric_name=="co2", "ppm",\
    metric_name=="power", "W",\
    metric_name=="voltage", "V",\
    metric_name=="current", "A",\
    1=1, "")

EVAL-metric_category = case(\
    metric_name IN ("temperature", "humidity", "pressure", "co2"), "environmental",\
    metric_name IN ("power", "voltage", "current"), "electrical",\
    1=1, "other")

EVAL-protocol = "mqtt"
EVAL-device_type = "iot_sensor"
EVAL-metric_quality = if(isnotnull('status') AND 'status'=="ok", "good", "suspect")
```

### Cisco Meraki (via MQTT/API)

```ini
[meraki_mt_json]
TIME_FORMAT = %Y-%m-%dT%H:%M:%SZ
SHOULD_LINEMERGE = false
LINE_BREAKER = ([\r\n]+)
KV_MODE = json

# Extract from MQTT topic
EXTRACT-mt_path = meraki/v1/mt/(?<network_id>[^/]+)/ble/(?<sensor_mac>[^/]+)/(?<metric_type_raw>[^/]+)$ in source

EVAL-device_id = "meraki_mt:".sensor_mac
EVAL-device_mac = sensor_mac
EVAL-organization_id = 'organizationId'
EVAL-network_id = network_id

# Map Meraki metric types
EVAL-metric_name = case(\
    metric_type_raw=="temperature", "temperature",\
    metric_type_raw=="humidity", "humidity",\
    metric_type_raw=="door", "door_state",\
    metric_type_raw=="waterDetection", "water_detected",\
    metric_type_raw=="CO2", "co2",\
    metric_type_raw=="tvoc", "voc",\
    metric_type_raw=="ambientNoise", "noise_level",\
    metric_type_raw=="PM2_5MassConcentration", "pm25",\
    metric_type_raw=="mainsRealPower", "power",\
    metric_type_raw=="batteryPercentage", "battery_level",\
    1=1, metric_type_raw)

EVAL-metric_value = coalesce(\
    tonumber('temperature'),\
    tonumber('humidity'),\
    tonumber('CO2'),\
    tonumber('tvoc'),\
    tonumber('ambientNoise'),\
    tonumber('PM2_5MassConcentration'),\
    tonumber('mainsRealPower'),\
    tonumber('batteryPercentage'))

EVAL-metric_unit = case(\
    metric_name=="temperature", "degC",\
    metric_name=="humidity", "%RH",\
    metric_name=="co2", "ppm",\
    metric_name=="voc", "ppb",\
    metric_name=="noise_level", "dBA",\
    metric_name=="pm25", "µg/m³",\
    metric_name=="power", "W",\
    metric_name=="battery_level", "percent",\
    1=1, "")

EVAL-protocol = "mqtt"
EVAL-vendor = "Cisco Meraki"
EVAL-device_type = "meraki_sensor"
EVAL-vertical = "facilities"
```

### Cisco Spaces (Firehose API)

```ini
[cisco_spaces_location]
TIME_FORMAT = %Y-%m-%dT%H:%M:%S.%3NZ
SHOULD_LINEMERGE = false
LINE_BREAKER = ([\r\n]+)
KV_MODE = json

# Device and location extraction
EVAL-tracked_device_id = 'device.device_id'
EVAL-device_mac = 'device.mac_address'
EVAL-tracked_device_type = lower('device.type')

# Location hierarchy
EVAL-location_id = 'location.location_id'
EVAL-location_name = 'location.name'
EVAL-location_type = lower('location.inferred_location_types{0}')
EVAL-site = 'location.parent.parent.name'
EVAL-building = 'location.parent.name'
EVAL-floor = if(location_type=="floor", 'location.name', null())

# Coordinates
EVAL-x_coordinate = tonumber('x_pos')
EVAL-y_coordinate = tonumber('y_pos')
EVAL-geo_lat = tonumber('latitude')
EVAL-geo_lon = tonumber('longitude')
EVAL-location_confidence = tonumber('confidence_factor')

# WiFi context
EVAL-ssid = 'ssid'
EVAL-ap_mac = 'associated_ap_mac'

EVAL-detection_method = "wifi"
EVAL-protocol = "spaces_api"
EVAL-vendor = "Cisco"
EVAL-device_type = "location_tracker"
EVAL-vertical = "facilities"
EVAL-system = "Location"
```

---

## Event/Alarm Configuration

### Mapping Alarms to Events Object

```ini
[your_alarm_sourcetype]
# Extract alarm fields
EXTRACT-alarm_code = alarm_code=(?<event_code>[^\s]+)
EXTRACT-alarm_msg = message="(?<event_message>[^"]+)"
EXTRACT-alarm_priority = priority=(?<alarm_priority>\d+)

# Map to Events object fields
EVAL-event_type = "alarm"

EVAL-event_severity = case(\
    alarm_priority<=1, "critical",\
    alarm_priority==2, "high",\
    alarm_priority==3, "medium",\
    alarm_priority>=4, "low",\
    1=1, "info")

EVAL-event_category = case(\
    match(event_code, "^TEMP"), "environmental",\
    match(event_code, "^ELEC"), "electrical",\
    match(event_code, "^MECH"), "equipment",\
    match(event_code, "^SEC"), "security",\
    1=1, "other")

EVAL-event_state = case(\
    match(_raw, "ACTIVE|ALARM|FAULT"), "active",\
    match(_raw, "CLEAR|NORMAL|RETURN"), "cleared",\
    match(_raw, "ACK"), "acknowledged",\
    1=1, "active")
```

---

## State Change Configuration

### Mapping to States Object

```ini
[your_state_sourcetype]
# Extract state changes
EXTRACT-state_change = object=(?<object_id>[^\s]+)\s+state=(?<new_state>\w+)\s+prev=(?<old_state>\w+)

EVAL-device_id = object_id

# Map to States object
EVAL-state_name = case(\
    match(object_id, "^PUMP"), "run_status",\
    match(object_id, "^VALVE"), "position",\
    match(object_id, "^DOOR"), "door_state",\
    match(object_id, "^MODE"), "operating_mode",\
    1=1, "status")

EVAL-state_value = lower(new_state)
EVAL-previous_state = lower(old_state)

EVAL-state_category = case(\
    state_name=="run_status", "operational",\
    state_name=="position", "position",\
    state_name=="door_state", "security",\
    state_name=="operating_mode", "mode",\
    1=1, "other")
```

---

## Production/OEE Configuration

### Manufacturing Data

```ini
[production_counts]
KV_MODE = json

EVAL-device_id = 'line_id'
EVAL-line_id = 'line_id'
EVAL-cell_id = 'cell_id'

# Production counts
EVAL-good_count = tonumber('counts.good')
EVAL-reject_count = tonumber('counts.reject')
EVAL-total_count = tonumber('counts.total')

# Timing
EVAL-cycle_time = tonumber('cycle_time_seconds')
EVAL-ideal_cycle_time = tonumber('ideal_cycle_seconds')

# Context
EVAL-shift = 'shift'
EVAL-product_code = 'product.code'
EVAL-batch_id = 'batch_id'
EVAL-operator_id = 'operator'

EVAL-vertical = "manufacturing"
EVAL-system = "Production"

[oee_data]
KV_MODE = json

EVAL-device_id = 'equipment_id'
EVAL-line_id = 'line_id'

# OEE components (as percentages 0-100)
EVAL-availability = tonumber('oee.availability')
EVAL-performance = tonumber('oee.performance')
EVAL-quality = tonumber('oee.quality')
EVAL-oee = tonumber('oee.overall')

# Downtime
EVAL-downtime_minutes = tonumber('downtime.duration_minutes')
EVAL-downtime_reason = 'downtime.reason'
EVAL-downtime_category = lower('downtime.category')

EVAL-vertical = "manufacturing"
EVAL-system = "OEE"
```

---

## Calculated Fields and Thresholds

### Using Lookups for Thresholds

```ini
[your_sourcetype]
# After extracting metric_name, apply threshold lookup
LOOKUP-thresholds = metric_thresholds metric_name OUTPUT metric_min metric_max low_critical low_warning high_warning high_critical

# Threshold status is calculated in the data model, but you can pre-compute:
EVAL-threshold_status = case(\
    metric_value < low_critical, "critical_low",\
    metric_value < low_warning, "warning_low",\
    metric_value > high_critical, "critical_high",\
    metric_value > high_warning, "warning_high",\
    1=1, "normal")
```

### Setpoint Deviation

```ini
[hvac_data]
# Extract setpoint from source or lookup
EVAL-metric_setpoint = coalesce(tonumber('setpoint'), tonumber('sp'))

# Deviation is calculated in data model, but can pre-compute:
EVAL-setpoint_deviation = if(isnotnull(metric_setpoint), metric_value - metric_setpoint, null())

# Source of setpoint
EVAL-setpoint_source = case(\
    isnotnull('override'), "operator",\
    isnotnull('schedule_sp'), "schedule",\
    isnotnull('optimization_sp'), "optimization",\
    1=1, "default")
```

---

## Validation Queries

After configuring your sourcetype, validate the mapping:

### Check Field Extraction

```spl
index=your_index sourcetype=your_sourcetype earliest=-1h
| stats count by device_id, metric_name, metric_value, metric_unit
| where isnotnull(device_id) AND isnotnull(metric_name) AND isnotnull(metric_value)
```

### Verify Data Model Mapping

```spl
| tstats count FROM datamodel=Operational_Telemetry 
  WHERE nodename=All_Telemetry.Metrics index=your_index 
  BY All_Telemetry.Metrics.metric_name, All_Telemetry.Metrics.metric_category
```

### Check for Missing Required Fields

```spl
index=your_index sourcetype=your_sourcetype earliest=-1h
| eval missing_fields = mvappend(
    if(isnull(device_id), "device_id", null()),
    if(isnull(device_type), "device_type", null()),
    if(isnull(vertical), "vertical", null()),
    if(isnull(site), "site", null()),
    if(isnull(system), "system", null()),
    if(isnull(metric_name), "metric_name", null()),
    if(isnull(metric_value), "metric_value", null()))
| where isnotnull(missing_fields)
| stats count by missing_fields
```

### Verify Lookup Enrichment

```spl
index=your_index sourcetype=your_sourcetype earliest=-1h
| stats count by device_id, site, building, floor, zone
| where isnull(site) OR isnull(building)
```

---

## Data Model Acceleration

### Configure in datamodels.conf

```ini
[Operational_Telemetry]
acceleration = 1
acceleration.earliest_time = -3mon
acceleration.backfill_time = -1d
acceleration.max_time = 60
acceleration.cron_schedule = */30 * * * *
tags = enabled
```

### Monitor Acceleration Health

```spl
| rest /services/admin/summarization/tstats:DM_Operational_Telemetry
| table title, summary.complete_time, summary.size, summary.last_error, summary.buckets_indexed
```

---

## Common Patterns

### Multi-Value Metric Handling

When a single event contains multiple metrics (common in JSON):

```ini
[multi_metric_json]
KV_MODE = json

# Create separate metric events using MV fields and mvexpand in searches
# OR use the first available metric (simpler but loses data):
EVAL-metric_name = case(\
    isnotnull('temp'), "temperature",\
    isnotnull('humidity'), "humidity",\
    isnotnull('pressure'), "pressure",\
    1=1, null())

EVAL-metric_value = coalesce(tonumber('temp'), tonumber('humidity'), tonumber('pressure'))
```

**Better approach using transforms.conf for multi-value extraction:**

```ini
# In transforms.conf
[extract_temperature]
REGEX = "temperature"\s*:\s*(?<metric_value__temperature>[\d.-]+)
FORMAT = metric_name::temperature metric_value::$1

[extract_humidity]  
REGEX = "humidity"\s*:\s*(?<metric_value__humidity>[\d.-]+)
FORMAT = metric_name::humidity metric_value::$1
```

### Handling Unit Conversion

```ini
[sensor_with_fahrenheit]
# Convert Fahrenheit to Celsius for consistency
EVAL-raw_temp_f = tonumber('temperature_f')
EVAL-metric_value = if(isnotnull(raw_temp_f), round((raw_temp_f - 32) * 5/9, 2), null())
EVAL-metric_unit = "degC"
EVAL-metric_name = "temperature"
```

### Handling State vs Metric

Some values can be either state or metric depending on context:

```ini
[door_sensor]
# Door position can be state (open/closed) or metric (angle)
EVAL-state_name = if(match('door_status', "open|closed"), "door_state", null())
EVAL-state_value = if(isnotnull(state_name), 'door_status', null())

EVAL-metric_name = if(isnum(tonumber('door_angle')), "door_position", null())
EVAL-metric_value = if(isnotnull(metric_name), tonumber('door_angle'), null())
EVAL-metric_unit = if(isnotnull(metric_name), "degrees", null())
```

---

## Troubleshooting

### Data Not Appearing in Data Model

1. **Check index constraint**: Ensure your index is in the model's search constraint
2. **Check required fields**: Verify `device_id`, `device_type`, `vertical`, `site`, `system` are populated
3. **Check object constraints**: For Metrics, verify `metric_name=* metric_value=*`
4. **Check acceleration**: Run `| datamodel Operational_Telemetry All_Telemetry search` directly

### Field Extraction Not Working

1. **Check sourcetype assignment**: `index=your_index | stats count by sourcetype`
2. **Check props.conf syntax**: Use `btool props list your_sourcetype --debug`
3. **Check field visibility**: Some fields may need `INDEXED_EXTRACTIONS` or `SHOULD_LINEMERGE = false`

### Lookup Not Enriching

1. **Check lookup file exists**: `| inputlookup your_lookup.csv | head 5`
2. **Check key field match**: Verify case sensitivity and exact match
3. **Check transforms.conf**: Ensure lookup is defined correctly
4. **Check automatic lookup**: Verify `LOOKUP-` directive in props.conf

---

## File Reference

| File | Purpose |
|------|---------|
| `props.conf` | Field extractions, aliases, calculated fields per sourcetype |
| `transforms.conf` | Lookup definitions, complex extractions |
| `datamodels.conf` | Data model acceleration settings |
| `Operational_Telemetry.json` | Data model definition (in `data/models/`) |
| `lookups/*.csv` | Enrichment lookup tables |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01 | Initial release with universal Metrics pattern |
| 1.1 | 2025-01 | Added OTI Cloud support (BMS, Meraki MT/MV, Edge Hub) |
| 1.2 | 2025-01 | Added Location object for Cisco Spaces |
| 2.0 | 2025-02 | Renamed from Industrial_Telemetry to Operational_Telemetry |

---

## Complete Field Reference

### All_Telemetry (Base Object) - 34 Fields

| Field | Type | Required | Description | Example Values |
|-------|------|----------|-------------|----------------|
| `device_id` | string | ✓ | Unique device identifier | `modbus:1:40001`, `meraki_mt:Q2XX-XXXX` |
| `device_name` | string | | Human-friendly name | `AHU-1 Supply Sensor` |
| `device_type` | string | ✓ | Device category | `plc`, `sensor`, `meter`, `controller`, `camera`, `gateway` |
| `device_mac` | string | | MAC address | `00:18:0a:xx:xx:xx` |
| `parent_device_id` | string | | Parent in hierarchy | `AHU-1` (for VAV controller) |
| `tag_path` | string | | Full hierarchical path | `ns=2;s=Building1.AHU1.SupplyTemp` |
| `asset_id` | string | | CMMS/EAM asset ID | `ASSET-001234` |
| `asset_name` | string | | Asset name | `Chiller Plant 1` |
| `asset_class` | string | | Asset classification | `rotating_equipment`, `electrical`, `hvac` |
| `vertical` | string | ✓ | Industry vertical | `manufacturing`, `facilities`, `smart_city`, `events` |
| `organization_id` | string | | Meraki org ID | `123456` |
| `network_id` | string | | Meraki network ID | `L_123456789` |
| `site` | string | ✓ | Site/campus code | `SITE_A`, `HQ`, `PLANT_1` |
| `site_name` | string | | Site full name | `Main Campus Building A` |
| `building` | string | | Building ID | `BLDG_1`, `Tower_A` |
| `floor` | string | | Floor/level | `1`, `B1`, `Roof` |
| `zone` | string | | Zone/area | `Zone_A`, `North_Wing` |
| `zone_type` | string | | Zone classification | `office`, `conference`, `lobby`, `mechanical` |
| `system` | string | ✓ | System category | `HVAC`, `Electrical`, `Production`, `Security` |
| `subsystem` | string | | Specific subsystem | `AHU-1`, `Chiller_Plant`, `Line_1` |
| `protocol` | string | | Communication protocol | `modbus`, `bacnet`, `opcua`, `mqtt` |
| `bacnet_object_type` | string | | BACnet object type | `AI`, `AO`, `AV`, `BI`, `BO`, `BV` |
| `bacnet_instance` | number | | BACnet instance number | `1`, `100`, `50001` |
| `vendor` | string | | Manufacturer | `Siemens`, `Cisco`, `Honeywell` |
| `model` | string | | Model number | `S7-1500`, `MT20`, `Tridium` |
| `serial_number` | string | | Serial number | `SN123456789` |
| `firmware_version` | string | | Firmware version | `v2.1.3` |
| `criticality` | string | | Criticality level | `critical`, `high`, `medium`, `low` |
| `geo_lat` | number | | GPS latitude | `51.5074` |
| `geo_lon` | number | | GPS longitude | `-0.1278` |
| `x_coordinate` | number | | Indoor X position | `125.5` |
| `y_coordinate` | number | | Indoor Y position | `340.2` |
| `location_confidence` | number | | Position accuracy | `0.95` (percentage) or `3.5` (meters) |
| `floor_plan_id` | string | | Floor plan reference | `FP_BLDG1_FL2` |

### Metrics Object - 16 Fields

| Field | Type | Required | Description | Example Values |
|-------|------|----------|-------------|----------------|
| `metric_name` | string | ✓ | Standardized metric name | `temperature`, `power`, `flow_rate` |
| `metric_value` | number | ✓ | Numeric measurement | `23.5`, `1500`, `0.85` |
| `metric_unit` | string | | Unit of measurement | `degC`, `kW`, `m3/h` |
| `metric_quality` | string | | Data quality indicator | `good`, `suspect`, `bad`, `stale` |
| `metric_min` | number | | Low operating limit | `15` |
| `metric_max` | number | | High operating limit | `30` |
| `metric_setpoint` | number | | Target/setpoint value | `22` |
| `setpoint_source` | string | | Setpoint origin | `schedule`, `operator`, `optimization` |
| `metric_category` | string | | Metric classification | `environmental`, `electrical`, `process` |
| `metric_dimension` | string | | Multi-axis indicator | `x`, `y`, `z`, `magnitude` |
| `aggregation_type` | string | | How to aggregate | `avg`, `sum`, `min`, `max`, `last` |
| `aggregation_period` | string | | Aggregation interval | `1min`, `5min`, `15min`, `1hour` |
| `lane_id` | string | | Traffic lane ID | `NB_1`, `SB_2` |
| `direction` | string | | Traffic direction | `northbound`, `inbound` |
| `tariff_id` | string | | Energy tariff ID | `TARIFF_001` |
| `rate_period` | string | | Rate period | `peak`, `off_peak`, `shoulder` |

**Calculated Fields** (auto-computed):
- `threshold_status`: `case(metric_value<metric_min,"low",metric_value>metric_max,"high",1=1,"normal")`
- `setpoint_deviation`: `if(isnotnull(metric_setpoint),metric_value-metric_setpoint,null())`

### Events Object - 14 Fields

| Field | Type | Required | Description | Example Values |
|-------|------|----------|-------------|----------------|
| `event_type` | string | ✓ | Event classification | `alarm`, `fault`, `warning`, `info` |
| `event_code` | string | | Alarm/fault code | `ALM-001`, `FAULT-HI-TEMP` |
| `event_message` | string | | Human-readable message | `High temperature alarm` |
| `event_severity` | string | | Severity level | `critical`, `high`, `medium`, `low`, `info` |
| `event_category` | string | | Event category | `equipment`, `environmental`, `security`, `process` |
| `event_state` | string | | Current state | `active`, `acknowledged`, `cleared` |
| `event_source` | string | | Source system | `BMS`, `SCADA`, `PLC` |
| `event_priority` | number | | Numeric priority | `1` (highest) to `5` (lowest) |
| `acknowledged` | boolean | | Ack status | `true`, `false` |
| `acknowledged_by` | string | | Who acknowledged | `jsmith`, `AUTO` |
| `acknowledged_time` | timestamp | | When acknowledged | `2025-01-15T10:30:00Z` |
| `cleared_time` | timestamp | | When cleared | `2025-01-15T11:00:00Z` |
| `duration_seconds` | number | | Time to clear | `1800` |
| `annotation` | string | | Operator notes | `Investigated - normal operation` |

### States Object - 10 Fields

| Field | Type | Required | Description | Example Values |
|-------|------|----------|-------------|----------------|
| `state_name` | string | ✓ | State identifier | `run_status`, `mode`, `position` |
| `state_value` | string | ✓ | Current state value | `running`, `auto`, `open` |
| `previous_state` | string | | Prior state value | `stopped`, `manual`, `closed` |
| `state_category` | string | | State category | `operational`, `position`, `mode` |
| `state_duration` | number | | Time in state (seconds) | `3600` |
| `transition_count` | number | | State changes count | `5` |
| `control_mode` | string | | Control method | `auto`, `manual`, `off`, `override` |
| `schedule_id` | string | | Active schedule | `SCHED_WEEKDAY` |
| `schedule_mode` | string | | Schedule state | `occupied`, `unoccupied`, `holiday` |
| `dwell_time` | number | | Time at location (sec) | `300` |

### Production Object - 18 Fields

| Field | Type | Required | Description | Example Values |
|-------|------|----------|-------------|----------------|
| `production_order` | string | | Production/work order | `PO-2025-001234` |
| `batch_id` | string | | Batch identifier | `BATCH-A001` |
| `product_code` | string | | Product SKU | `PROD-XYZ-100` |
| `product_name` | string | | Product name | `Widget Assembly` |
| `line_id` | string | | Production line | `LINE_1`, `CELL_A` |
| `cell_id` | string | | Work cell | `WELD_CELL_1` |
| `shift` | string | | Work shift | `Day`, `Night`, `1`, `2`, `3` |
| `operator_id` | string | | Operator badge | `OP-12345` |
| `good_count` | number | | Good parts produced | `150` |
| `reject_count` | number | | Rejected parts | `3` |
| `total_count` | number | | Total parts | `153` |
| `scrap_count` | number | | Scrapped parts | `2` |
| `rework_count` | number | | Reworked parts | `1` |
| `cycle_time` | number | | Actual cycle (seconds) | `45.2` |
| `ideal_cycle_time` | number | | Target cycle (seconds) | `40.0` |
| `takt_time` | number | | Required rate (seconds) | `42.0` |
| `recipe_version` | string | | Recipe/program version | `R2.1` |
| `program_id` | string | | PLC/CNC program | `PROG_001` |

### OEE Object - 11 Fields

| Field | Type | Required | Description | Example Values |
|-------|------|----------|-------------|----------------|
| `oee` | number | | Overall OEE percentage | `85.5` |
| `availability` | number | | Availability % | `92.0` |
| `performance` | number | | Performance % | `95.0` |
| `quality` | number | | Quality % | `98.0` |
| `planned_production_time` | number | | Planned time (minutes) | `480` |
| `actual_run_time` | number | | Actual run (minutes) | `442` |
| `downtime_minutes` | number | | Total downtime | `38` |
| `downtime_reason` | string | | Downtime cause | `Changeover`, `Breakdown` |
| `downtime_category` | string | | Downtime class | `planned`, `unplanned` |
| `speed_loss_minutes` | number | | Speed losses | `15` |
| `yield_rate` | number | | First-pass yield % | `97.5` |

### Quality Object - 15 Fields

| Field | Type | Required | Description | Example Values |
|-------|------|----------|-------------|----------------|
| `measurement_name` | string | | SPC measurement | `diameter`, `weight`, `thickness` |
| `measurement_value` | number | | Measured value | `25.02` |
| `measurement_unit` | string | | Unit | `mm`, `g`, `µm` |
| `nominal` | number | | Target value | `25.00` |
| `usl` | number | | Upper spec limit | `25.10` |
| `lsl` | number | | Lower spec limit | `24.90` |
| `ucl` | number | | Upper control limit | `25.05` |
| `lcl` | number | | Lower control limit | `24.95` |
| `in_spec` | boolean | | Within spec? | `true`, `false` |
| `in_control` | boolean | | Within control? | `true`, `false` |
| `inspection_type` | string | | Inspection method | `inline`, `offline`, `final` |
| `sample_id` | string | | Sample identifier | `SAMPLE-001` |
| `sample_size` | number | | Sample quantity | `5` |
| `defect_code` | string | | Defect classification | `DEF-001` |
| `defect_count` | number | | Number of defects | `2` |

### Maintenance Object - 20 Fields

| Field | Type | Required | Description | Example Values |
|-------|------|----------|-------------|----------------|
| `work_order_id` | string | | Work order number | `WO-2025-001` |
| `work_order_type` | string | | WO classification | `preventive`, `corrective`, `predictive` |
| `work_order_status` | string | | Current status | `open`, `in_progress`, `completed` |
| `work_order_priority` | string | | Priority level | `emergency`, `urgent`, `normal`, `low` |
| `failure_code` | string | | Failure classification | `FAIL-BEARING-001` |
| `failure_description` | string | | Failure details | `Bearing overheating` |
| `technician_id` | string | | Assigned tech | `TECH-001` |
| `labor_hours` | number | | Hours worked | `4.5` |
| `parts_cost` | number | | Parts expense | `250.00` |
| `total_cost` | number | | Total expense | `450.00` |
| `scheduled_date` | timestamp | | Scheduled date | `2025-02-01T08:00:00Z` |
| `completed_date` | timestamp | | Completion date | `2025-02-01T12:30:00Z` |
| `runtime_hours` | number | | Equipment runtime | `8760` |
| `runtime_since_maintenance` | number | | Hours since PM | `720` |
| `health_score` | number | | Health 0-100 | `85.5` |
| `remaining_useful_life` | number | | RUL in hours/days | `2160` |
| `failure_probability` | number | | Failure prob 0-1 | `0.15` |
| `anomaly_score` | number | | Anomaly 0-100 | `25.3` |
| `prediction_model` | string | | ML model used | `bearing_failure_v2` |
| `prediction_confidence` | number | | Model confidence | `0.92` |

### Security Object - 14 Fields

| Field | Type | Required | Description | Example Values |
|-------|------|----------|-------------|----------------|
| `access_point` | string | | Door/gate ID | `DOOR-MAIN-001` |
| `access_direction` | string | | Entry/exit | `entry`, `exit` |
| `access_result` | string | | Access outcome | `granted`, `denied`, `tailgate` |
| `credential_type` | string | | Credential method | `card`, `pin`, `biometric`, `mobile` |
| `credential_id` | string | | Badge/credential # | `BADGE-12345` |
| `person_id` | string | | Employee/visitor ID | `EMP-001` |
| `person_type` | string | | Person category | `employee`, `visitor`, `contractor` |
| `camera_id` | string | | Camera identifier | `CAM-LOBBY-001` |
| `intrusion_zone` | string | | Security zone | `ZONE-PERIMETER-1` |
| `visitor_company` | string | | Visitor's company | `ACME Corp` |
| `host_id` | string | | Visitor's host | `EMP-002` |
| `ticket_id` | string | | Event ticket ID | `TKT-2025-001234` |
| `ticket_type` | string | | Ticket category | `general`, `vip`, `season`, `staff` |
| `ticket_tier` | string | | Pricing tier | `premium`, `standard`, `economy` |

### VenueOperations Object - 12 Fields

| Field | Type | Required | Description | Example Values |
|-------|------|----------|-------------|----------------|
| `event_id` | string | | Event identifier | `EVT-2025-001` |
| `event_name` | string | | Event name | `Championship Game` |
| `event_start` | timestamp | | Start time | `2025-06-15T19:00:00Z` |
| `event_end` | timestamp | | End time | `2025-06-15T22:00:00Z` |
| `venue_id` | string | | Venue identifier | `STADIUM-001` |
| `venue_name` | string | | Venue name | `City Arena` |
| `venue_capacity` | number | | Maximum capacity | `50000` |
| `gate_id` | string | | Entry gate | `GATE-A` |
| `section_id` | string | | Seating section | `SEC-101` |
| `concession_id` | string | | Concession stand | `CONC-NORTH-1` |
| `parking_lot_id` | string | | Parking area | `LOT-A` |
| `route_id` | string | | Transit route | `BUS-SHUTTLE-1` |

### Location Object - 11 Fields

| Field | Type | Required | Description | Example Values |
|-------|------|----------|-------------|----------------|
| `location_id` | string | | Spaces location UUID | `loc-uuid-xxx` |
| `location_name` | string | | Location name | `Conference Room A` |
| `location_type` | string | | Location level | `campus`, `building`, `floor`, `zone` |
| `tracked_device_id` | string | | Device being tracked | `DEV-uuid-xxx` |
| `tracked_device_type` | string | | Device category | `mobile`, `tag`, `client`, `asset` |
| `ssid` | string | | WiFi network | `Corporate-WiFi` |
| `ap_mac` | string | | Access point MAC | `00:18:0a:xx:xx:xx` |
| `detection_method` | string | | Tracking method | `wifi`, `ble`, `gps`, `rfid` |
| `visit_id` | string | | Visit/journey ID | `VISIT-uuid-xxx` |
| `entry_time` | timestamp | | Zone entry time | `2025-01-15T09:00:00Z` |
| `exit_time` | timestamp | | Zone exit time | `2025-01-15T10:30:00Z` |

---

## Index Configuration

### Recommended Index Structure

Create indexes in `indexes.conf`:

```ini
# Industrial IoT primary index
[industrial_iot]
homePath = $SPLUNK_DB/industrial_iot/db
coldPath = $SPLUNK_DB/industrial_iot/colddb
thawedPath = $SPLUNK_DB/industrial_iot/thaweddb
maxDataSize = auto_high_volume
frozenTimePeriodInSecs = 7776000

# Manufacturing-specific
[manufacturing]
homePath = $SPLUNK_DB/manufacturing/db
coldPath = $SPLUNK_DB/manufacturing/colddb
thawedPath = $SPLUNK_DB/manufacturing/thaweddb

# BMS/Facilities
[bms]
homePath = $SPLUNK_DB/bms/db
coldPath = $SPLUNK_DB/bms/colddb
thawedPath = $SPLUNK_DB/bms/thaweddb

# Edge Hub data (high volume)
[edge_hub_mqtt]
homePath = $SPLUNK_DB/edge_hub_mqtt/db
coldPath = $SPLUNK_DB/edge_hub_mqtt/colddb
thawedPath = $SPLUNK_DB/edge_hub_mqtt/thaweddb
maxDataSize = auto_high_volume

[edge_hub_ot]
homePath = $SPLUNK_DB/edge_hub_ot/db
coldPath = $SPLUNK_DB/edge_hub_ot/colddb
thawedPath = $SPLUNK_DB/edge_hub_ot/thaweddb
```

### Adding New Indexes to Data Model

Update the constraint in `Operational_Telemetry.json`:

```json
"constraints": [
    {
        "search": "index=industrial OR index=iot OR index=manufacturing OR index=bms OR index=scada OR index=your_new_index"
    }
]
```

---

## SPL Query Examples

### Basic Queries Using tstats (Accelerated)

```spl
# Count metrics by name (last 24h)
| tstats count FROM datamodel=Operational_Telemetry 
  WHERE nodename=All_Telemetry.Metrics 
  BY All_Telemetry.Metrics.metric_name

# Average temperature by site
| tstats avg(All_Telemetry.Metrics.metric_value) AS avg_temp 
  FROM datamodel=Operational_Telemetry 
  WHERE nodename=All_Telemetry.Metrics 
    All_Telemetry.Metrics.metric_name=temperature 
  BY All_Telemetry.site

# Active alarms count by severity
| tstats count FROM datamodel=Operational_Telemetry 
  WHERE nodename=All_Telemetry.Events 
    All_Telemetry.Events.event_state=active 
  BY All_Telemetry.Events.event_severity

# OEE by production line
| tstats avg(All_Telemetry.OEE.oee) AS oee,
    avg(All_Telemetry.OEE.availability) AS availability,
    avg(All_Telemetry.OEE.performance) AS performance,
    avg(All_Telemetry.OEE.quality) AS quality
  FROM datamodel=Operational_Telemetry 
  WHERE nodename=All_Telemetry.OEE 
  BY All_Telemetry.Production.line_id
```

### Time-Series Analysis

```spl
# Hourly temperature trend
| tstats avg(All_Telemetry.Metrics.metric_value) AS avg_temp 
  FROM datamodel=Operational_Telemetry 
  WHERE nodename=All_Telemetry.Metrics 
    All_Telemetry.Metrics.metric_name=temperature 
  BY _time, All_Telemetry.site span=1h
| timechart span=1h avg(avg_temp) BY All_Telemetry.site

# Equipment state duration
| tstats count, sum(All_Telemetry.States.state_duration) AS total_duration 
  FROM datamodel=Operational_Telemetry 
  WHERE nodename=All_Telemetry.States 
  BY All_Telemetry.device_id, All_Telemetry.States.state_value
```

### Threshold Violation Detection

```spl
# Find metrics exceeding thresholds
| tstats latest(All_Telemetry.Metrics.metric_value) AS value,
    latest(All_Telemetry.Metrics.metric_min) AS min_threshold,
    latest(All_Telemetry.Metrics.metric_max) AS max_threshold
  FROM datamodel=Operational_Telemetry 
  WHERE nodename=All_Telemetry.Metrics 
  BY All_Telemetry.device_id, All_Telemetry.Metrics.metric_name
| where value < min_threshold OR value > max_threshold
| eval status=case(value<min_threshold, "LOW", value>max_threshold, "HIGH")
```

### Cross-Object Correlation

```spl
# Correlate alarms with equipment states
| tstats count AS alarm_count 
  FROM datamodel=Operational_Telemetry 
  WHERE nodename=All_Telemetry.Events 
    All_Telemetry.Events.event_severity IN (critical, high)
  BY All_Telemetry.device_id, _time span=1h
| join type=left All_Telemetry.device_id 
  [| tstats latest(All_Telemetry.States.state_value) AS state 
     FROM datamodel=Operational_Telemetry 
     WHERE nodename=All_Telemetry.States 
       All_Telemetry.States.state_name=run_status 
     BY All_Telemetry.device_id]
```

---

## Macro Definitions

Add these to `macros.conf` for reusable queries:

```ini
[ot_metrics(2)]
args = metric_name, timespan
definition = | tstats avg(All_Telemetry.Metrics.metric_value) AS value FROM datamodel=Operational_Telemetry WHERE nodename=All_Telemetry.Metrics All_Telemetry.Metrics.metric_name=$metric_name$ BY _time, All_Telemetry.site span=$timespan$

[ot_active_alarms]
definition = | tstats count FROM datamodel=Operational_Telemetry WHERE nodename=All_Telemetry.Events All_Telemetry.Events.event_state=active BY All_Telemetry.Events.event_severity, All_Telemetry.site

[ot_oee_summary]
definition = | tstats avg(All_Telemetry.OEE.oee) AS oee, avg(All_Telemetry.OEE.availability) AS availability, avg(All_Telemetry.OEE.performance) AS performance, avg(All_Telemetry.OEE.quality) AS quality FROM datamodel=Operational_Telemetry WHERE nodename=All_Telemetry.OEE BY All_Telemetry.Production.line_id

[ot_device_health]
definition = | tstats latest(All_Telemetry.Maintenance.health_score) AS health, latest(All_Telemetry.Maintenance.failure_probability) AS fail_prob FROM datamodel=Operational_Telemetry WHERE nodename=All_Telemetry.Maintenance BY All_Telemetry.device_id, All_Telemetry.device_name
```

Usage:
```spl
`ot_metrics(temperature, 1h)`
`ot_active_alarms`
`ot_oee_summary`
```

---

## Saved Searches and Alerts

Add to `savedsearches.conf`:

```ini
[OT - Critical Alarm Alert]
search = | tstats count FROM datamodel=Operational_Telemetry WHERE nodename=All_Telemetry.Events All_Telemetry.Events.event_severity=critical All_Telemetry.Events.event_state=active BY All_Telemetry.device_id, All_Telemetry.Events.event_message | where count > 0
dispatch.earliest_time = -5m
dispatch.latest_time = now
cron_schedule = */5 * * * *
enableSched = 1
alert.severity = 5
alert.track = 1
alert_condition = search count > 0
action.email = 1
action.email.to = ops-team@example.com
action.email.subject = CRITICAL: OT Alarm Detected

[OT - Equipment Health Degradation]
search = | tstats latest(All_Telemetry.Maintenance.health_score) AS health FROM datamodel=Operational_Telemetry WHERE nodename=All_Telemetry.Maintenance BY All_Telemetry.device_id, All_Telemetry.device_name | where health < 70 AND health > 0
dispatch.earliest_time = -1h
dispatch.latest_time = now
cron_schedule = 0 * * * *
enableSched = 1
alert.severity = 3
alert_condition = search count > 0

[OT - Data Model Acceleration Health]
search = | rest /services/admin/summarization by_tstats=t splunk_server=local | search title="*Operational_Telemetry*" | table title, summary.complete_time, summary.size, summary.is_inprogress, summary.last_error
dispatch.earliest_time = -1h
dispatch.latest_time = now
cron_schedule = 0 6 * * *
enableSched = 1

[OT - Missing Data Detection]
search = | tstats count FROM datamodel=Operational_Telemetry WHERE nodename=All_Telemetry BY All_Telemetry.device_id, _time span=1h | streamstats window=2 current=f last(count) AS prev_count BY All_Telemetry.device_id | where count=0 AND prev_count>0
dispatch.earliest_time = -2h
dispatch.latest_time = now
cron_schedule = */15 * * * *
enableSched = 1
alert.severity = 2
```

---

## Dashboard Best Practices

### Recommended tstats Patterns for Dashboards

```xml
<!-- Single Value - Total Devices -->
<search>
  <query>| tstats dc(All_Telemetry.device_id) AS count FROM datamodel=Operational_Telemetry WHERE nodename=All_Telemetry</query>
  <earliest>$time.earliest$</earliest>
  <latest>$time.latest$</latest>
</search>

<!-- Timechart - Metric Trend -->
<search>
  <query>| tstats avg(All_Telemetry.Metrics.metric_value) AS value FROM datamodel=Operational_Telemetry WHERE nodename=All_Telemetry.Metrics All_Telemetry.Metrics.metric_name=$metric_name$ BY _time span=auto | timechart span=auto avg(value)</query>
  <earliest>$time.earliest$</earliest>
  <latest>$time.latest$</latest>
</search>

<!-- Table - Active Alarms -->
<search>
  <query>| tstats count, latest(_time) AS last_time FROM datamodel=Operational_Telemetry WHERE nodename=All_Telemetry.Events All_Telemetry.Events.event_state=active BY All_Telemetry.device_name, All_Telemetry.Events.event_severity, All_Telemetry.Events.event_message | sort -last_time | eval last_time=strftime(last_time, "%Y-%m-%d %H:%M:%S")</query>
  <earliest>$time.earliest$</earliest>
  <latest>$time.latest$</latest>
</search>
```

### Performance Tips

1. **Always use tstats** for accelerated data model searches
2. **Filter early** - put most restrictive constraints first
3. **Use span=auto** for timecharts to let Splunk optimize
4. **Limit BY clauses** - fewer fields = faster queries
5. **Use prestats=true** for very large datasets

---

## Appendix: Complete props.conf Template

```ini
#############################################################################
# OPERATIONAL TELEMETRY - COMPLETE SOURCETYPE TEMPLATE
# Copy and customize for your data source
#############################################################################

[your_sourcetype_name]
# =========================================================================
# PARSING CONFIGURATION
# =========================================================================
TIME_FORMAT = %Y-%m-%dT%H:%M:%S.%3NZ
TIME_PREFIX = "timestamp"\s*:\s*"
MAX_TIMESTAMP_LOOKAHEAD = 40
SHOULD_LINEMERGE = false
LINE_BREAKER = ([\r\n]+)
TRUNCATE = 10000
KV_MODE = json

# =========================================================================
# BASE FIELDS (All_Telemetry) - REQUIRED
# =========================================================================
# Device identification
EVAL-device_id = coalesce('sensor_id', 'device.id', 'id')
EVAL-device_name = coalesce('sensor_name', 'device.name', 'name')
EVAL-device_type = coalesce('sensor_type', 'device.type', "sensor")
EVAL-device_mac = 'mac_address'

# Location hierarchy
EVAL-site = coalesce('location.site', 'site_id', "DEFAULT")
EVAL-site_name = 'location.site_name'
EVAL-building = 'location.building'
EVAL-floor = 'location.floor'
EVAL-zone = 'location.zone'

# Classification
EVAL-vertical = "facilities"
EVAL-system = "HVAC"
EVAL-subsystem = 'equipment_id'
EVAL-protocol = "mqtt"

# Equipment hierarchy (optional)
EVAL-asset_id = 'asset.id'
EVAL-asset_name = 'asset.name'
EVAL-vendor = 'device.manufacturer'
EVAL-model = 'device.model'
EVAL-criticality = coalesce('priority', "medium")

# =========================================================================
# METRICS OBJECT FIELDS
# =========================================================================
# Determine metric_name from available fields
EVAL-metric_name = case(\
    isnotnull('temperature'), "temperature",\
    isnotnull('humidity'), "humidity",\
    isnotnull('pressure'), "pressure",\
    isnotnull('co2'), "co2",\
    isnotnull('power'), "power",\
    isnotnull('voltage'), "voltage",\
    isnotnull('current'), "current",\
    isnotnull('flow_rate'), "flow_rate",\
    isnotnull('level'), "level",\
    isnotnull('count'), "count",\
    1=1, null())

# Extract metric value
EVAL-metric_value = coalesce(\
    tonumber('temperature'),\
    tonumber('humidity'),\
    tonumber('pressure'),\
    tonumber('co2'),\
    tonumber('power'),\
    tonumber('voltage'),\
    tonumber('current'),\
    tonumber('flow_rate'),\
    tonumber('level'),\
    tonumber('count'),\
    tonumber('value'))

# Map to standard units
EVAL-metric_unit = case(\
    metric_name=="temperature", "degC",\
    metric_name=="humidity", "%RH",\
    metric_name=="pressure", "kPa",\
    metric_name=="co2", "ppm",\
    metric_name=="power", "kW",\
    metric_name=="voltage", "V",\
    metric_name=="current", "A",\
    metric_name=="flow_rate", "m3/h",\
    metric_name=="level", "percent",\
    metric_name=="count", "count",\
    1=1, coalesce('unit', 'units', ""))

# Categorize metrics
EVAL-metric_category = case(\
    metric_name IN ("temperature","humidity","pressure","co2","voc","pm25"), "environmental",\
    metric_name IN ("power","voltage","current","energy","frequency"), "electrical",\
    metric_name IN ("flow_rate","level","pressure"), "process",\
    metric_name IN ("count","occupancy"), "performance",\
    1=1, "other")

# Quality indicator
EVAL-metric_quality = case(\
    isnotnull('status') AND lower('status')=="ok", "good",\
    isnotnull('status') AND lower('status')=="error", "bad",\
    isnotnull('quality'), lower('quality'),\
    1=1, "good")

# Thresholds (from data or lookup)
EVAL-metric_setpoint = tonumber('setpoint')
EVAL-metric_min = tonumber('low_limit')
EVAL-metric_max = tonumber('high_limit')

# =========================================================================
# EVENTS OBJECT FIELDS (for alarm/fault sourcetypes)
# =========================================================================
EVAL-event_type = case(\
    match(_raw, "ALARM|alarm"), "alarm",\
    match(_raw, "FAULT|fault"), "fault",\
    match(_raw, "WARN|warn"), "warning",\
    1=1, "info")

EVAL-event_code = 'alarm_code'
EVAL-event_message = coalesce('alarm_message', 'message', 'description')

EVAL-event_severity = case(\
    'priority'<=1 OR match(_raw, "CRITICAL|critical"), "critical",\
    'priority'==2 OR match(_raw, "HIGH|high"), "high",\
    'priority'==3 OR match(_raw, "MEDIUM|medium"), "medium",\
    1=1, "low")

EVAL-event_state = case(\
    match(_raw, "ACTIVE|active|ALARM"), "active",\
    match(_raw, "CLEAR|clear|NORMAL|RTN"), "cleared",\
    match(_raw, "ACK|ack"), "acknowledged",\
    1=1, "active")

# =========================================================================
# STATES OBJECT FIELDS (for state-change sourcetypes)
# =========================================================================
EVAL-state_name = case(\
    isnotnull('run_status'), "run_status",\
    isnotnull('mode'), "mode",\
    isnotnull('position'), "position",\
    isnotnull('door_state'), "door_state",\
    1=1, null())

EVAL-state_value = lower(coalesce('run_status', 'mode', 'position', 'door_state', 'state'))

EVAL-state_category = case(\
    state_name=="run_status", "operational",\
    state_name IN ("mode", "control_mode"), "mode",\
    state_name=="position", "position",\
    state_name=="door_state", "security",\
    1=1, "other")

# =========================================================================
# LOOKUPS FOR ENRICHMENT
# =========================================================================
LOOKUP-device_enrichment = device_metadata device_id OUTPUT device_name device_type site site_name building floor zone system subsystem asset_id criticality
LOOKUP-thresholds = metric_thresholds metric_name OUTPUT metric_min metric_max low_critical low_warning high_warning high_critical
```
