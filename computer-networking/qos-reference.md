# QoS (Quality of Service) Deep Reference

## DSCP / Per-Hop Behavior Complete Table

### Class Selector (CS) — Backward compatible with IP Precedence

| PHB | DSCP Name | Decimal | Binary | IP Precedence | Typical Use |
|-----|-----------|---------|--------|---------------|-------------|
| CS0 | CS0 (DF/BE) | 0 | 000000 | 0 (Routine) | Best effort / default |
| CS1 | CS1 | 8 | 001000 | 1 (Priority) | Scavenger / bulk |
| CS2 | CS2 | 16 | 010000 | 2 (Immediate) | OAM (network management) |
| CS3 | CS3 | 24 | 011000 | 3 (Flash) | Broadcast video / signaling |
| CS4 | CS4 | 32 | 100000 | 4 (Flash Override) | Real-time interactive |
| CS5 | CS5 | 40 | 101000 | 5 (Critical) | Signaling (SIP, H.323) |
| CS6 | CS6 | 48 | 110000 | 6 (Internetwork Control) | Network control (routing protocols) |
| CS7 | CS7 | 56 | 111000 | 7 (Network Control) | Reserved |

### Assured Forwarding (AF) Classes

DSCP = 8 * Class + 2 * Drop Precedence

| Class | Low Drop (1) | Medium Drop (2) | High Drop (3) | Typical Use |
|-------|-------------|-----------------|---------------|-------------|
| AF1x | AF11 = 10 | AF12 = 12 | AF13 = 14 | Bulk data (backup, file transfer) |
| AF2x | AF21 = 18 | AF22 = 20 | AF23 = 22 | Transactional data (ERP, CRM) |
| AF3x | AF31 = 26 | AF32 = 28 | AF33 = 30 | Multimedia streaming |
| AF4x | AF41 = 34 | AF42 = 36 | AF43 = 38 | Interactive video (conferencing) |

### Expedited Forwarding (EF)

| PHB | DSCP | Decimal | Use |
|-----|------|---------|-----|
| EF | EF | 46 | Voice bearer (strict priority) |
| VOICE-ADMIT | VA | 44 | Admitted voice (CAC-controlled) |

### Cisco QoS Baseline (RFC 4594 / Cisco 12-class model)

| Class | DSCP | PHB | Queue | Use Case |
|-------|------|-----|-------|----------|
| Voice | EF (46) | EF | Priority (LLQ) | VoIP bearer |
| Broadcast Video | CS5 (40) | CS5 | Priority (LLQ) | Live video |
| Real-Time Interactive | CS4 (32) | CS4 | Priority (LLQ) | Video conferencing |
| Multimedia Conferencing | AF41 (34) | AF4 | Bandwidth guarantee | Interactive video |
| Multimedia Streaming | AF31 (26) | AF3 | Bandwidth guarantee | Streaming video |
| Signaling | CS3 (24) | CS3 | Bandwidth guarantee | SIP, SCCP, H.323 |
| Transactional Data | AF21 (18) | AF2 | Bandwidth guarantee | ERP, CRM, database |
| OAM | CS2 (16) | CS2 | Bandwidth guarantee | SNMP, syslog, NTP |
| Bulk Data | AF11 (10) | AF1 | Bandwidth guarantee | FTP, backup, email |
| Scavenger | CS1 (8) | CS1 | Minimum bandwidth | P2P, gaming, personal |
| Best Effort | DF (0) | BE | Default | Everything else |
| Network Control | CS6 (48) | CS6 | Bandwidth guarantee | OSPF, BGP, EIGRP, HSRP |

---

## Queuing Mechanisms

### Priority Queuing (PQ)
- Strict priority; high-priority queue always served first
- Risk: lower queues can starve
- Used ONLY for delay-sensitive traffic (voice, real-time video)
- Always combine with a policer to prevent starvation

### Weighted Fair Queuing (WFQ)
- Divides bandwidth fairly among flows based on IP precedence
- Automatic; no manual class configuration
- Legacy; replaced by CBWFQ in modern networks

### Class-Based Weighted Fair Queuing (CBWFQ)
- Assign guaranteed minimum bandwidth per class
- Classes defined by class-map (match criteria)
- Each class gets a dedicated queue
- Bandwidth can be specified in kbps, percentage, or remaining percentage
- Unmatched traffic goes to default queue

### Low-Latency Queuing (LLQ)
- CBWFQ + strict priority queue for voice/video
- `priority` command in policy-map creates strict priority queue with built-in policer
- Priority traffic gets served first, up to configured rate; excess is dropped (policed)
- Prevents starvation of other queues

### Weighted Random Early Detection (WRED)
- Congestion avoidance (not management)
- Randomly drops packets before queue is full
- Drop probability based on DSCP/IP Precedence
- Prevents TCP global synchronization
- Applied within CBWFQ classes

---

## QoS Policy Configuration (Cisco IOS-XE)

### MQC (Modular QoS CLI) Structure
```
class-map match-all VOICE
 match dscp ef
class-map match-all VIDEO
 match dscp af41
class-map match-all SIGNALING
 match dscp cs3
class-map match-all TRANSACTIONAL
 match dscp af21
class-map match-all BULK
 match dscp af11
class-map match-all SCAVENGER
 match dscp cs1
class-map match-all NET-CTRL
 match dscp cs6
!
policy-map WAN-EDGE
 class VOICE
  priority percent 10
 class VIDEO
  priority percent 23
 class SIGNALING
  bandwidth percent 2
 class TRANSACTIONAL
  bandwidth percent 15
  random-detect dscp-based
 class BULK
  bandwidth percent 5
  random-detect dscp-based
 class NET-CTRL
  bandwidth percent 3
 class SCAVENGER
  bandwidth percent 1
 class class-default
  bandwidth percent 25
  random-detect dscp-based
  fair-queue
!
interface GigabitEthernet0/0
 service-policy output WAN-EDGE
```

---

## Policing vs Shaping

| Aspect | Policing | Shaping |
|--------|---------|---------|
| Action on excess | Drop or re-mark | Buffer and delay |
| Direction | Inbound or outbound | Outbound only |
| Burst handling | Tc (committed burst) + Be (excess burst) | Bc (committed burst) per Tc interval |
| Latency impact | None (drops immediately) | Adds delay (buffering) |
| Use case | Service provider edge; ingress classification | WAN edge; conform to circuit CIR |
| Token bucket | Single-rate or dual-rate | Token replenished per interval |

### Single-Rate Three-Color Policer (srTCM)
- Bc (Committed Burst): traffic up to CIR → Conform (green)
- Be (Excess Burst): traffic between CIR and CIR+Be → Exceed (yellow)
- Above both → Violate (red)
- Actions: transmit, drop, set-dscp-transmit

### Two-Rate Three-Color Policer (trTCM)
- CIR + PIR (Peak Information Rate)
- Below CIR → Conform; between CIR and PIR → Exceed; above PIR → Violate

### Cisco Policing Example
```
policy-map POLICE-100M
 class class-default
  police cir 100000000 bc 3125000 be 6250000
   conform-action transmit
   exceed-action drop
   violate-action drop
```

### Cisco Shaping Example
```
policy-map SHAPE-100M
 class class-default
  shape average 100000000 3125000
  service-policy WAN-EDGE
```

---

## QoS Marking and Trust

### Trust Boundaries
- **Access layer**: Trust boundary at switch port; mark DSCP at ingress
- **Untrusted ports**: Re-mark to CS0 (best effort) unless trusted
- **Cisco IP Phone**: Trust DSCP from phone; re-mark PC traffic behind phone
- **Wireless**: Mark at WLC or AP; re-mark at switch if needed

### Marking Locations
| Location | Mark With | Trust |
|----------|-----------|-------|
| End host | DSCP (if trusted) | Configured per port |
| Access switch | DSCP via MQC | Trust boundary |
| Distribution/Core | Trust existing DSCP | Pass through |
| WAN edge | Verify/re-mark; apply policy | Trust internal |
| Service provider | Often re-marks to their scheme | Contract-dependent |

### CoS to DSCP Mapping (Default)

| CoS (802.1p) | IP Precedence | Default DSCP |
|--------------|--------------|-------------|
| 0 | 0 | 0 (BE) |
| 1 | 1 | 8 (CS1) |
| 2 | 2 | 16 (CS2) |
| 3 | 3 | 24 (CS3) |
| 4 | 4 | 32 (CS4) |
| 5 | 5 | 40 (CS5) |
| 6 | 6 | 48 (CS6) |
| 7 | 7 | 56 (CS7) |

---

## QoS Design Models

### RFC 4594 (DiffServ Service Classes)
The authoritative reference for DSCP-to-service mapping; defines 12 service classes.

### Cisco SRND (Solution Reference Network Design)
- 4-class: Voice, Video, Data, Best Effort (minimum)
- 8-class: Adds Signaling, OAM, Scavenger, Bulk
- 12-class: Full model as shown above (recommended)

### Design Rules of Thumb
1. **Priority queue**: Never exceed 33% of link bandwidth (voice + video combined)
2. **Voice**: Always EF (46); LLQ priority; 150 ms one-way delay budget
3. **Video**: AF41 or CS4; separate from voice; dedicated bandwidth
4. **Signaling**: CS3; small bandwidth but guaranteed (2-5%)
5. **Network control**: CS6; always guaranteed bandwidth (3-5%); never drop routing protocol packets
6. **Default**: At minimum 25% of bandwidth for best effort
7. **Mark close to source**: Set DSCP at access layer, not in the core
8. **End-to-end**: QoS must be configured on EVERY hop in the path
