# Cabling & Physical Layer Deep Reference

## Copper Cabling Standards

### Category Specifications

| Category | Standard | Max Frequency | Max Speed | Max Distance | Shielding | Use Case |
|----------|----------|-------------|----------|-------------|-----------|----------|
| Cat5e | TIA-568-C.2 | 100 MHz | 1 Gbps | 100m | UTP | Legacy; avoid for new |
| Cat6 | TIA-568-C.2 | 250 MHz | 1G (100m) / 10G (55m) | 100m / 55m | UTP or STP | SMB; short 10G runs |
| Cat6a | TIA-568-C.2 | 500 MHz | 10 Gbps | 100m | STP or UTP | Enterprise standard |
| Cat7 | ISO/IEC 11801 | 600 MHz | 10 Gbps | 100m | S/FTP | European; not TIA recognized |
| Cat7a | ISO/IEC 11801 | 1000 MHz | 10 Gbps | 100m | S/FTP | European; not TIA recognized |
| Cat8.1 | TIA-568-C.2-1 | 2000 MHz | 25G/40 Gbps | 30m | STP | Data center short runs |
| Cat8.2 | ISO/IEC 11801 | 2000 MHz | 25G/40 Gbps | 30m | S/FTP | Data center short runs |

**Notes:**
- Cat6 10G distance: 55m unshielded, 37m in some conditions; varies by alien crosstalk
- Cat6a is the recommended standard for new enterprise installations (supports WiFi 6E/7 AP backhaul at 10G, PoE++, and future 25G BASE-T)
- Cat8 is NOT for horizontal cabling; designed for data center top-of-rack connections only
- Solid conductor for permanent runs; stranded for patch cables
- Maximum channel length: 100m total = 90m permanent link + 10m patch cords

### Ethernet Standards Over Copper

| Standard | Speed | Cable | Max Distance | Pairs Used | PoE Compatible |
|----------|-------|-------|-------------|-----------|----------------|
| 100BASE-TX | 100 Mbps | Cat5+ | 100m | 2 pairs | Yes |
| 1000BASE-T | 1 Gbps | Cat5e+ | 100m | 4 pairs | Yes |
| 2.5GBASE-T | 2.5 Gbps | Cat5e+ | 100m | 4 pairs | Yes |
| 5GBASE-T | 5 Gbps | Cat6+ | 100m | 4 pairs | Yes |
| 10GBASE-T | 10 Gbps | Cat6a (Cat6: 55m) | 100m (Cat6a) | 4 pairs | Yes |
| 25GBASE-T | 25 Gbps | Cat8 | 30m | 4 pairs | No |
| 40GBASE-T | 40 Gbps | Cat8 | 30m | 4 pairs | No |

### Connector Types (Copper)
| Connector | Use | Standard |
|-----------|-----|----------|
| RJ-45 (8P8C) | Ethernet | All Cat5e-Cat6a |
| TERA | Cat7/7a | ISO/IEC 61076-3-104 |
| ARJ45/GG45 | Cat7/7a backward compatible | ISO/IEC 60603-7 |
| RJ-45 Cat8 | Cat8.1 | TIA-568-C.2-1 |

### T568A vs T568B Wiring
| Pin | T568A | T568B |
|-----|-------|-------|
| 1 | White/Green | White/Orange |
| 2 | Green | Orange |
| 3 | White/Orange | White/Green |
| 4 | Blue | Blue |
| 5 | White/Blue | White/Blue |
| 6 | Orange | Green |
| 7 | White/Brown | White/Brown |
| 8 | Brown | Brown |

- **Straight-through**: Same standard both ends (T568B most common in US)
- **Crossover**: T568A one end, T568B other end (rarely needed; Auto-MDIX handles it)
- **Auto-MDIX**: Automatic crossover detection; standard on all modern switches

---

## Fiber Optic Cabling

### Multimode Fiber Types

| Type | Core/Cladding | Bandwidth | 1G Distance | 10G Distance | 40G Distance | 100G Distance | Color |
|------|--------------|-----------|-------------|-------------|-------------|--------------|-------|
| OM1 | 62.5/125 um | 200 MHz·km | 275m (SX) | 33m | — | — | Orange |
| OM2 | 50/125 um | 500 MHz·km | 550m (SX) | 82m | — | — | Orange |
| OM3 | 50/125 um (laser-optimized) | 2000 MHz·km | 550m (SX) | 300m (SR) | 100m (SR4) | 100m (SR4) | Aqua |
| OM4 | 50/125 um (laser-optimized) | 4700 MHz·km | 550m (SX) | 400m (SR) | 150m (SR4) | 150m (SR4) | Aqua/Violet |
| OM5 | 50/125 um (wideband) | 4700+ MHz·km | 550m | 400m | 440m (SWDM4) | 150m (SR4) | Lime green |

**Notes:**
- OM1 and OM2 are legacy; do NOT specify for new installations (removed from TIA-568.3-E)
- OM3 minimum for 10G/40G/100G data center deployments
- OM4 is the standard for modern data centers
- OM5 adds wideband support for SWDM (Short Wavelength Division Multiplexing) — future-proofing for higher speeds on multimode

### Single-Mode Fiber Types

| Type | Core/Cladding | Wavelength | Max Distance | ITU-T | Use Case |
|------|--------------|------------|-------------|-------|----------|
| OS1 | 9/125 um | 1310/1550 nm | 10 km | G.652 | Campus/building |
| OS1a | 9/125 um | 1310/1550 nm | 10 km | G.652.D | Indoor; harmonized with ISO |
| OS2 | 9/125 um | 1310/1550 nm | 40-80 km+ | G.652.D/G.657 | Metro, DCI, long-haul |

**Notes:**
- OS2 is the standard for new single-mode installations
- OS1/OS1a for indoor/tight-bend applications
- Single-mode supports virtually unlimited bandwidth with coherent optics
- Bend-insensitive fiber (G.657.A1/A2/B3) recommended for indoor routing

### Fiber Connector Types

| Connector | Type | Ferrule | Use Case | Latching |
|-----------|------|---------|----------|----------|
| LC | Small form-factor | 1.25mm | SFP/SFP+ transceivers; most common | Push-pull |
| SC | Standard | 2.5mm | Legacy; some GPON | Push-pull |
| ST | Standard | 2.5mm | Legacy; older MM installations | Bayonet twist |
| MPO/MTP-12 | Multi-fiber | 12 fibers | 40G SR4, 100G SR4 | Push-pull |
| MPO/MTP-24 | Multi-fiber | 24 fibers | 100G SR10 | Push-pull |
| MPO/MTP-32 | Multi-fiber | 32 fibers | 400G SR8 | Push-pull |
| FC | Standard | 2.5mm | Telecom (rarely in DC) | Screw-on |
| E2000 | Standard | 1.25mm | European telecom | Spring-loaded |

**Polish types:**
- **UPC (Ultra Physical Contact)**: Blue; return loss ≥50 dB; standard for multimode
- **APC (Angled Physical Contact)**: Green; return loss ≥65 dB; required for single-mode in CATV, PON, analog systems
- **PC**: Standard; return loss ≥40 dB

NEVER mate APC with UPC/PC connectors — will damage the ferrule endface.

---

## Transceiver Types

### SFP/SFP+ Family

| Form Factor | Speed | Interface |
|------------|-------|-----------|
| SFP | 1G | GLC-SX-MMD, GLC-LH-SMD, GLC-T |
| SFP+ | 10G | SFP-10G-SR, SFP-10G-LR, SFP-10G-ER |
| SFP28 | 25G | SFP-25G-SR-S, SFP-25G-LR-S |
| SFP56 | 50G | SFP-50G-SR, SFP-50G-LR |

### QSFP Family

| Form Factor | Speed | Typical Use |
|------------|-------|-------------|
| QSFP+ | 40G | QSFP-40G-SR4, QSFP-40G-LR4 |
| QSFP28 | 100G | QSFP-100G-SR4-S, QSFP-100G-LR4, QSFP-100G-CWDM4 |
| QSFP56 | 200G | QSFP-200G-SR4, QSFP-200G-FR4 |
| QSFP-DD | 400G | QDD-400G-DR4-S, QDD-400G-FR4, QDD-400G-ZR-S |
| OSFP | 400G/800G | Alternative to QSFP-DD; larger thermal envelope |

### Common Transceiver Reach

| Optic | Speed | Fiber Type | Wavelength | Max Distance |
|-------|-------|-----------|------------|-------------|
| SX/SR | 1G/10G | OM3/OM4 MM | 850 nm | 300m/400m |
| LX/LR | 1G/10G | OS2 SM | 1310 nm | 10 km |
| EX/ER | 1G/10G | OS2 SM | 1550 nm | 40 km |
| ZX/ZR | 1G/10G | OS2 SM | 1550 nm | 80 km |
| SR4 | 40G/100G | OM3/OM4 MM | 850 nm (4x) | 100m/150m |
| LR4 | 40G/100G | OS2 SM | 1310 nm (4λ) | 10 km |
| DR4 | 100G/400G | OS2 SM | 1310 nm (4x) | 500m |
| FR4 | 100G/400G | OS2 SM | 1310 nm (4λ) | 2 km |
| CWDM4 | 100G | OS2 SM | 4 CWDM λ | 2 km |
| 400ZR | 400G | OS2 SM | C-band coherent | 120 km+ |
| 400ZR+ | 400G | OS2 SM | C-band coherent | 500+ km |
| GLC-T / SFP-10G-T | 1G/10G | Cat6a copper | N/A | 30m-100m |

### DAC (Direct Attach Copper)
- Passive DAC: up to 5m (10G), 3m (25G/100G)
- Active DAC: up to 10m (10G), 5m (25G/100G)
- Cheapest and lowest latency option for short rack connections
- No fiber or transceiver needed (cable has built-in SFP+/QSFP+ ends)

### AOC (Active Optical Cable)
- Pre-terminated fiber with built-in transceivers
- Distances: 1m to 100m typically
- Lower cost than separate optic + fiber for data center use
- Not field-terminable

---

## PoE (Power over Ethernet)

### PoE Standards

| Standard | IEEE | Max Power (PSE) | Max Power (PD) | Pairs | Voltage | Use Case |
|----------|------|----------------|----------------|-------|---------|----------|
| PoE | 802.3af | 15.4W | 12.95W | 2 | 48V | IP phones, basic cameras |
| PoE+ | 802.3at | 30W | 25.5W | 2 | 48V | Pan-tilt-zoom cameras, thin APs |
| UPoE / PoE++ (Type 3) | 802.3bt | 60W | 51W | 4 | 48V | WiFi 6 APs, video phones |
| UPoE+ / PoE++ (Type 4) | 802.3bt | 100W | 71.3W | 4 | 48V | WiFi 6E/7 APs, digital signage, thin clients |

**Notes:**
- Power loss over cable: plan for ~10-20% loss at 100m
- Minimum cable: Cat5e for all PoE standards; Cat6a recommended for Type 3/4 (better heat dissipation)
- PSE = Power Sourcing Equipment (switch); PD = Powered Device (AP, camera)
- Cisco UPOE delivers up to 60W; Cisco UPOE+ delivers up to 90W (pre-standard implementations)
- PoE budget per switch: check total available PoE watts; varies by model and power supply

### PoE Troubleshooting
```
show power inline              ! Cisco: show PoE status per port
show power inline detail       ! Cisco: detailed PoE info
show power inline consumption  ! Cisco: total power budget/usage
```

---

## Structured Cabling Design

### TIA-568 Standards
- **TIA-568.0-E**: Generic cabling standard
- **TIA-568.1-E**: Commercial building cabling
- **TIA-568.2-E**: Balanced twisted-pair cabling (copper)
- **TIA-568.3-E**: Optical fiber cabling
- **TIA-606-C**: Administration standard (labeling, documentation)
- **TIA-607-D**: Grounding and bonding

### Cabling Infrastructure
| Element | Description | Standard |
|---------|-------------|----------|
| Entrance Facility (EF) | Demarcation from service provider | TIA-568.1 |
| Equipment Room (ER) | Core/distribution switches, patch panels | TIA-568.1 |
| Telecommunications Room (TR) | Floor-level IDF; access switches | TIA-569 |
| Horizontal Cabling | TR to work area outlet; max 90m permanent | TIA-568.1 |
| Backbone Cabling | Between TRs and ER; fiber or copper | TIA-568.1 |
| Work Area | Outlet to device; patch cable max 10m | TIA-568.1 |

### Cable Management Best Practices
- Maintain bend radius: 4x cable diameter (copper), 10x cable diameter (fiber inside), 15x (fiber outside)
- Do not exceed cable weight/fill limits in cable trays
- Separate power and data cables (EMI); minimum 12" separation for unshielded
- Label both ends of every cable (TIA-606)
- Use cable managers in racks; avoid zip-ties on fiber (use velcro)
- Test all cable runs: Cat6a requires field certification (not just wire mapping)
