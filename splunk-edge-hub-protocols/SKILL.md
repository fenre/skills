---
name: splunk-edge-hub-protocols
description: >
  Industrial protocol configuration reference for Splunk Edge Hub. Use this skill when:
  (1) Configuring Modbus TCP/RTU connections — function codes, data types, byte order, register ranges;
  (2) Setting up OPC UA connections — URL format, security modes, node ID formats, subscription parameters;
  (3) Configuring MQTT broker connections — topic patterns, QoS levels, wildcards;
  (4) Working with SNMP polling — version support, common OIDs, community strings;
  (5) Setting up BACnet connections — object types, property identifiers, network ports.
---

# Edge Hub Protocol Configuration Reference

## Modbus TCP Configuration

### Connection Parameters
| Parameter | Description | Default |
|-----------|-------------|---------|
| `captureHostname` | Target device IP address | Required |
| `destPort` | Modbus TCP port | 502 |
| `deviceId` | Unit/slave ID (1-247) | 1 |
| `pollInterval` | Polling frequency (ms) | 1000 |
| `timeout` | Request timeout (ms) | 5000 |

### Function Codes
| Code | Function | Register Type |
|------|----------|---------------|
| 01 | `READ_COILS` | Discrete outputs |
| 02 | `READ_DISCRETE_INPUTS` | Discrete inputs |
| 03 | `READ_HOLDING_REGISTERS` | Read/write registers |
| 04 | `READ_INPUT_REGISTERS` | Read-only registers |

### Data Types (unitIdentifier)
| Type | Size | Range |
|------|------|-------|
| `UINT_16BIT` | 1 register | 0 to 65535 |
| `INT_16BIT` | 1 register | -32768 to 32767 |
| `UINT_32BIT` | 2 registers | 0 to 4294967295 |
| `INT_32BIT` | 2 registers | -2147483648 to 2147483647 |
| `FLOAT_32BIT` | 2 registers | IEEE 754 floating point |
| `FLOAT_64BIT` | 4 registers | IEEE 754 double precision |

### Byte Order Options
- `BIG_ENDIAN` (default): High byte first
- `LITTLE_ENDIAN`: Low byte first
- `BIG_ENDIAN_SWAP`: Big endian with word swap
- `LITTLE_ENDIAN_SWAP`: Little endian with word swap

---

## OPC UA Configuration

### Connection URL Format
```
opc.tcp://<hostname>:<port>/<server_path>
```

### Security Modes
| Mode | Description |
|------|-------------|
| `None` | No encryption (testing only) |
| `Sign` | Message signing |
| `SignAndEncrypt` | Full encryption |

### Node ID Formats
| Format | Example |
|--------|---------|
| Numeric | `ns=2;i=1234` |
| String | `ns=2;s=Channel1.Device1.Tag1` |
| GUID | `ns=2;g=12345678-1234-1234-1234-123456789abc` |
| Opaque | `ns=2;b=M/RbKBsRVkePCePcx24oRA==` |

### Subscription Parameters
| Parameter | Description | Default |
|-----------|-------------|---------|
| `samplingInterval` | Data collection rate (ms) | 1000 |
| `publishingInterval` | Update push rate (ms) | 1000 |
| `queueSize` | Buffered values | 10 |

---

## MQTT Configuration

### Broker Connection
| Parameter | Description |
|-----------|-------------|
| `broker_url` | MQTT broker address (tcp:// or ssl://) |
| `port` | Broker port (1883 TCP, 8883 TLS) |
| `client_id` | Unique client identifier |
| `username` | Authentication username |
| `password` | Authentication password |

### Topic Subscription Patterns
| Pattern | Matches |
|---------|---------|
| `sensors/+/temperature` | Single-level wildcard |
| `sensors/#` | Multi-level wildcard |
| `meraki/v1/mt/+/ble/#` | Meraki MT sensors |
| `/merakimv/+/0` | Meraki camera zone 0 |

### QoS Levels
| Level | Guarantee |
|-------|-----------|
| 0 | At most once (fire and forget) |
| 1 | At least once (acknowledged) |
| 2 | Exactly once (four-way handshake) |

---

## SNMP Configuration

### Version Support
| Version | Features |
|---------|----------|
| SNMPv1 | Basic, community string auth |
| SNMPv2c | Improved, 64-bit counters |
| SNMPv3 | Authentication + encryption |

### Common OIDs
| OID | Description |
|-----|-------------|
| `.1.3.6.1.2.1.1.1.0` | System description |
| `.1.3.6.1.2.1.1.3.0` | System uptime |
| `.1.3.6.1.2.1.2.2.1.*` | Interface statistics |

---

## BACnet Configuration

### Object Types
| Type ID | Object Type |
|---------|-------------|
| 0 | Analog Input |
| 1 | Analog Output |
| 2 | Analog Value |
| 3 | Binary Input |
| 4 | Binary Output |
| 5 | Binary Value |
| 13 | Multi-state Input |
| 14 | Multi-state Output |

### Property Identifiers
| ID | Property |
|----|----------|
| 85 | Present Value |
| 77 | Object Name |
| 28 | Description |
| 117 | Units |

### Network Ports
| Protocol | Port |
|----------|------|
| BACnet/IP | 47808 (UDP) |
| BACnet/MSTP | Serial |
