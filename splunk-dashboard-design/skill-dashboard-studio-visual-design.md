# SKILL: Splunk Dashboard Studio — Visual Design Patterns

## Purpose
This skill defines the visual design language, layout patterns, and component anatomy for building production-quality Splunk Dashboard Studio dashboards. It is derived from proven dashboard implementations across multiple verticals and serves as the authoritative reference for both React prototyping and Dashboard Studio JSON generation.

## Scope
- Visual design patterns and color systems
- Layout architectures and grid patterns
- KPI card anatomy and variants
- Chart and visualization styling
- Background treatments and decorative elements
- React-to-Studio translation mapping
- Vertical-specific design guidelines

---

## 1. DESIGN PRINCIPLES

### 1.1 Visual Hierarchy
- **Primary KPIs**: Largest type (48-72px equivalent), placed in the top row or hero position
- **Secondary KPIs**: Medium type (24-36px), grouped in supporting cards
- **Tertiary Metrics**: Small type (14-18px), used for trend deltas, labels, sub-metrics
- **Trend Indicators**: Always paired with primary/secondary KPIs — arrow icon + delta value + color coding

### 1.2 Information Density
- Dashboards should feel **data-rich but not cluttered**
- Use container grouping (rounded rectangles, bordered sections) to create visual breathing room
- Maximum 4 KPI cards per row at desktop width
- Charts should occupy at least 300px height to remain readable
- White space is used **between groups**, not within groups

### 1.3 Consistency Rules
- All KPI cards within a section use the same anatomy (value + unit + trend + sparkline)
- Chart types are consistent within a comparison group (don't mix bar and line for same-level metrics)
- Color coding for severity/status is universal across all dashboards (see Section 3)
- Trend arrows always use the same iconography: ↑ (up) ↓ (down) with color context

---

## 2. LAYOUT ARCHITECTURES

### 2.1 Two-Column Split (e.g., Home Care Management Overview)
```
┌─────────────────────────┬─────────────────────────┐
│      SECTION HEADER     │      SECTION HEADER     │
├────────────┬────────────┼────────────┬────────────┤
│  KPI Card  │  KPI Card  │  KPI Card  │  KPI Card  │
├────────────┴────────────┼────────────┴────────────┤
│     Time-series Chart   │     Time-series Chart   │
├────────────┬────────────┼─────┬──────┬────────────┤
│  KPI Card  │  KPI Card  │ KPI │ KPI  │  KPI Card  │
├────────────┴────────────┼─────┴──────┴────────────┤
│     Time-series Chart   │     Time-series Chart   │
└─────────────────────────┴─────────────────────────┘
```
**Use when:** Two distinct operational domains share a single overview (e.g., Patient Care + Operations)
**Studio implementation:** Full-width canvas, two equal-width groups, absolute positioning within each group

### 2.2 Left Detail + Right Context (e.g., Home Care Patient Overview)
```
┌──────────────────────────────────────────────────────────┐
│  VITAL HEALTH METRICS (horizontal KPI strip)             │
├──────────────────────────┬───────────────────────────────┤
│   LIVING METRICS         │   CARE PLAN                   │
│   (3x3 sensor grid)      │   (schedule + medication      │
│                          │    table + care steps)         │
│   + Assisted Living      │   + Critical Care Info         │
│     Requirements Table   │     (narrative text block)     │
└──────────────────────────┴───────────────────────────────┘
```
**Use when:** Entity-level drill-down with mixed data types (sensors, tables, text)
**Key feature:** Top strip spans full width for vital signs with sparklines

### 2.3 Hub-and-Spoke Topology (e.g., Operation Panels / Cloud Health)
```
                    ┌───────────┐
                    │  AWS East │
                    │  Score+KPIs│
┌───────────┐       └─────┬─────┘       ┌───────────┐
│ AWS West  │─────────────┼─────────────│ Azure UK  │
│ Score+KPIs│       ┌─────┴─────┐       │ Score+KPIs│
└───────────┘       │ HQ On-prem│       └───────────┘
                    │ Score+KPIs│
                    └─────┬─────┘       ┌───────────┐
                          └─────────────│ GCP Tokyo │
                                        │ Score+KPIs│
                                        └───────────┘
```
**Use when:** Distributed infrastructure health, multi-site operations
**Studio implementation:** Absolute positioning with connecting lines via `viz.markdown` SVG overlays
**Key detail:** Latency badges on connection lines (e.g., "87ms ↓-3" in colored pill)

### 2.4 Production Flow (e.g., Manufacturing Production Overview)
```
┌──────────────┐    ════►    ┌──────────────┐    ◄════    ┌──────────────┐
│ Body Assembly│              │  Shifts Hub  │              │Rotors Assembly│
│  OEE + Score │              │  3 shift %s  │              │  SL + Score  │
├──────────────┤              │  + Model Prod│              ├──────────────┤
│ Temp/Vib/PSI │              │  + Chart     │              │ Temp/Vib/PSI │
├──────────────┤              └──────────────┘              ├──────────────┤
│Incident Table│                                           │Incident Table│
└──────────────┘                                           └──────────────┘
┌─────┬─────┬─────┬─────┬─────┬──────────┬────────────────┐
│ OEE │Avail│Perf │Qual │Energy│Prod Cost │ Op Risk        │
└─────┴─────┴─────┴─────┴─────┴──────────┴────────────────┘
```
**Use when:** Manufacturing lines, process flows, sequential operations
**Key feature:** Directional flow arrows between stations, mirrored layout for parallel lines, summary KPI strip at bottom

### 2.5 Command Center (e.g., Smart Facilities Overview)
```
┌──────────────┬────────────────────┬───────────────┬──────────────────┐
│ Monthly Cost │   Daily Cost Chart │  Monthly Occ  │ Daily Occ Chart  │
├──────────────┴──────┬─────────────┴───┬───────────┴──────────────────┤
│                     │  KPI Compliance │                              │
│  Incident List      │   (radial gauge)│  Sub-KPI Gauges (6 donuts)  │
│                     │                 │                              │
├─────────────────────┴─────────────────┴──────────────────────────────┤
│                    BUILDING ILLUSTRATION                             │
│              (SVG/HTML background with weather overlays)             │
└──────────────────────────────────────────────────────────────────────┘
```
**Use when:** Facilities management, campus/building operations
**Key feature:** Illustrative building graphic as hero visual, KPI gauges in orbital/radial arrangement

### 2.6 Domain Dashboard (e.g., Security Overview, Customer Engagement)
```
┌─────────────────────────────────────────────────────────────────┐
│  HEADER: Logo + Title + Time Picker + Date/Time                 │
├────────────┬──────────────────────────┬─────────────────────────┤
│ Severity   │    Geographic Map /      │  Threat / Category      │
│ Stack      │    Hero Visualization    │  Breakdown              │
│ (3 cards)  │                          │  (bar chart)            │
├────────────┼──────────┬───────────────┼─────────────────────────┤
│ Risk Level │          │               │  Device / Entity        │
│ (big num)  │  Security Posture Hub   │  Counts                 │
├────────────┤  (radial with satellite  │  (hexagon cards)        │
│ Timeline   │   program scores)        │                         │
│ Chart      │          │               │  Response + SLA         │
└────────────┴──────────┴───────────────┴─────────────────────────┘
```
**Use when:** Security operations, domain-specific command centers
**Key feature:** Central posture/score hub with satellite program metrics, severity color coding throughout

---

## 3. COLOR SYSTEMS

### 3.1 Light Mode — Healthcare / Home Care
```
Background:          #E8F8F0 (soft mint)
Card Background:     #4ECB8D (medium green)
Card Text:           #FFFFFF
Primary KPI:         #1A1A2E (dark navy) or #FFFFFF on green
Trend Up (positive): #2ECC71 (green) 
Trend Down (negative): #E74C3C (red)
High Risk/Alert:     #E74C3C (red)
Warning:             #F39C12 (amber)
Chart Bar Primary:   #2980B9 (blue)
Chart Bar Alert:     #C0392B (dark red)
Section Header:      #1A1A2E (dark navy, bold)
```

### 3.2 Dark Mode — Business / Operations / Security
```
Background:          #0A0E27 (deep navy) to #141832 (dark blue)
Card Background:     #1A1E3A (dark card) with subtle border
Card Text:           #FFFFFF
Primary KPI:         #FFFFFF (large) 
Secondary KPI:       #8892B0 (muted blue-grey)
Accent Blue:         #64FFDA or #00BCD4 (teal/cyan)
Accent Green:        #4ECB8D
Trend Up (positive): #4ECB8D (green)
Trend Down (negative): #FF6B6B (coral red)
Critical/High:       #FF4757 (bright red)
Warning:             #FFA502 (orange)
Chart Gain:          #4ECB8D (green)
Chart Loss:          #FF6B6B (red)
Chart Neutral:       #5B6DCD (purple-blue)
Sparkline:           #64FFDA (cyan) on dark bg
```

### 3.3 Dark Mode — Security Specific
```
Background:          #0D1117 (near black) with radial gradient overlay
Severity High:       #FF4757 with pulsing glow
Severity Medium:     #FFA502 (amber/orange)
Severity Low:        #00BCD4 (cyan)
Security Posture Hub:#8B0000 (dark red gradient) for degraded, #2ECC71 for healthy
Threat Bars:         #FF6B9D (hot pink/coral)
Map Background:      #1A1E3A (dark) with #2A2E4A landmasses
Map Pins:            #FF4757 (red)
Hexagon Cards:       #0D2137 border with cyan accent
```

### 3.4 Cyan Mode — Facilities / Smart Building
```
Background:          #00CED1 (bright cyan/turquoise)
Card Background:     #008B8B (darker teal) or #006666
Card Border:         #004D4D (dark teal)
Text Primary:        #FFFFFF
Text Secondary:      #E0FFFF (light cyan)
Alert/Incident:      #FF4757 (red on cyan — high contrast)
Warning:             #FFD700 (gold)
OK/Normal:           #98FB98 (pale green)
Chart Colors:        Multi-city palette (pink, blue, green, orange, red)
Gauge Ring:          #4ECB8D (green fill) on #2A2E4A (dark track)
Building Illustration: Flat design, #87CEEB sky, #228B22 grass, neutral building tones
```

### 3.5 Severity / Status Universal Colors
```
Critical:    #FF4757 (bright red)
High:        #FF6B6B (coral)
Medium/Warn: #FFA502 (amber/orange)  
Low/Info:    #00BCD4 (cyan) or #5DADE2 (light blue)
OK/Online:   #4ECB8D (green)
Degraded:    #F39C12 (dark amber)
Offline:     #95A5A6 (grey)
```

---

## 4. KPI CARD ANATOMY

### 4.1 Standard KPI Card
```
┌─────────────────────────────┐
│ Label (small, uppercase)     │
│                              │
│  1,234  ↑ 56                │
│  ██████ (unit)  (delta)     │
│  [sparkline ~~~~~~~~]        │
└─────────────────────────────┘
```
**Components:**
- Label: 12-14px, uppercase or title case, lighter color
- Value: 36-56px, bold, primary color
- Unit: 14-18px, appended to value (e.g., "bpm", "%", "kWh", "ms")
- Trend Arrow: ↑ or ↓ icon, colored green (good) or red (bad) — NOTE: direction meaning is context-dependent (e.g., ↑ risk = bad, ↑ uptime = good)
- Delta: 14-18px, same color as arrow
- Sparkline: Optional, 60-100px wide, thin line showing recent trend

### 4.2 Score Card (Radial Gauge)
```
┌─────────────────────────────┐
│       Section Label          │
│    ┌─────────────────┐      │
│    │    ╭───────╮     │      │
│    │    │  87   │     │      │
│    │    ╰───────╯     │      │
│    └─────────────────┘      │
│    Sub-metric  Sub-metric    │
└─────────────────────────────┘
```
**Use when:** Overall health scores, OEE, security posture, compliance
**Studio mapping:** `viz.radialGauge` or custom SVG in `viz.markdown`

### 4.3 Status Card
```
┌─────────────────────────────┐
│  Server status               │
│                              │
│    Online                    │
│    (large, green, bold)      │
└─────────────────────────────┘
```
**Use when:** Binary or categorical status (Online/Offline, Active/Inactive)
**Color:** Full background color matches status

### 4.4 Multi-Metric Card
```
┌─────────────────────────────┐
│  Body Assembly Details       │
│  Temperature  Vibration  PSI│
│  🌡 80°F    (w) 88Hz   ⚙ 22│
└─────────────────────────────┘
```
**Use when:** Sensor clusters, environmental readings, grouped sub-metrics
**Key:** Icons + compact values, no sparklines, tight horizontal layout

### 4.5 Comparison Strip (Horizontal KPI Row)
```
┌──────┬──────┬──────┬──────┬──────┐
│ OEE  │Avail │ Perf │ Qual │Energy│
│ 57%  │ 65%  │ 109% │ 98%  │1.42K │
│ ↓-1  │ ↓-1  │ ↑15  │ ↑7   │ ↑50  │
└──────┴──────┴──────┴──────┴──────┘
```
**Use when:** Summary footer, cross-cutting KPIs, benchmark comparisons
**Studio:** Row of `viz.singlevalue` panels with minimal padding

---

## 5. CHART STYLING PATTERNS

### 5.1 Time-Series Bar Chart
- Bar width: ~70-80% of available slot
- Bar color: Solid single color (blue for neutral, red for alert data)
- Y-axis: Abbreviated (K, M) with max 4-5 gridlines
- X-axis: Date labels, rotated if needed, show day-of-week for daily granularity
- Background: Transparent (inherits card/section background)

### 5.2 Stacked Bar Chart (Multi-Category)
- Use distinct colors per category (not shades of same color)
- Legend: Top-aligned, horizontal, color swatches
- Max 5 categories before using "Other"

### 5.3 Sparklines (In-Card)
- Height: 30-50px
- Color: Single color matching the metric's accent
- No axes, no labels — pure trend shape
- Line weight: 1.5-2px
- Area fill: Optional subtle gradient (10-20% opacity)

### 5.4 Donut / Radial Gauge
- Track color: Dark muted (#2A2E4A on dark, #E0E0E0 on light)
- Fill color: Status-appropriate (green for healthy, red for critical)
- Center text: Score value, large bold
- Stroke width: 10-15px for gauges, 20-30px for donuts
- Satellite metrics: Positioned around the donut in orbital pattern

---

## 6. DECORATIVE & ADVANCED ELEMENTS

### 6.1 Background Images
- **Building illustrations** (Image 3): Flat-design SVG, positioned bottom half of dashboard
- **Photographic backgrounds** (Image 7): Dark overlay (70-80% opacity) to maintain text readability
- **Gradient backgrounds**: Radial gradient from center (slightly lighter) to edges (darker)
- **Implementation:** `viz.markdown` panel at z-index 0, full canvas width/height, or canvas background option

### 6.2 Section Containers
- Rounded rectangles (8-12px radius) with:
  - Subtle border (1-2px, slightly lighter than background)
  - Slightly different background shade than canvas
  - Section title inside top-left
- **Implementation:** `viz.markdown` with styled div, positioned behind content panels

### 6.3 Connecting Lines & Flow Arrows
- Used in topology views and production flows
- Dashed or solid lines, 2-3px weight
- Arrow heads for directional flow
- Latency/metric badges on lines (pill-shaped, colored)
- **Implementation:** SVG overlay in `viz.markdown` panel

### 6.4 Icons & Logos
- Use inline SVG or emoji for metric icons (🌡 ⚙ 🔒 📊)
- Vendor logos (AWS, Azure, GCP, Cisco) via `viz.markdown` with `<img>` tags or inline SVG
- Dashboard logo/branding: Top-left corner, 40-60px height
- **Implementation:** `viz.markdown` panels with HTML/SVG content

### 6.5 Geographic Maps
- Dark-themed world/region map as background
- Pin markers for locations (colored by severity)
- **Implementation:** `viz.choropleth` or custom SVG map in `viz.markdown`

---

## 7. HEADER PATTERNS

### 7.1 Standard Dashboard Header
```
┌──────────────────────────────────────────────────────────────┐
│ [Logo] App Name    Dashboard Title    [Time Picker]  Date Time│
└──────────────────────────────────────────────────────────────┘
```
- Logo: 40-50px, left-aligned
- App name: 14-16px, muted color
- Dashboard title: 24-32px, bold, primary color
- Time picker: Center or center-right
- Current date/time: Right-aligned, muted

### 7.2 With Filters/Dropdowns
```
┌──────────────────────────────────────────────────────────────┐
│ [Logo] Dashboard Title                    [Filter 1] [Filter 2]│
│                                           Edge Hub: TV-DVT2   │
│                                           Patient: Mary L.     │
└──────────────────────────────────────────────────────────────┘
```
- Filters right-aligned or below header
- Dropdown styling: Dark input fields on dark bg, light on light bg

---

## 8. VERTICAL-SPECIFIC GUIDELINES

### 8.1 Healthcare / Home Care
- **Color mode:** Light (mint green)
- **Priority metrics:** Patient count, risk levels, device health, staff active
- **Special elements:** Care plan tables, medication tables, assisted living requirements (text blocks)
- **Sensitivity:** Use warm, approachable colors; avoid aggressive red except for genuine high-risk
- **Living metrics grid:** 3-column sensor layout (temp, humidity, noise, IAQ, CO2, light, room visits)

### 8.2 Smart Facilities / Building Management
- **Color mode:** Cyan/turquoise OR dark mode
- **Priority metrics:** Cost, occupancy, KPI compliance, incident count
- **Special elements:** Building illustration, weather overlay, multi-site comparison charts
- **Sub-KPI gauges:** Orbital arrangement around central compliance score (Room Usage, Utilities, Infrastructure, Lighting, Environmental, Networking)

### 8.3 Business Operations / E-Commerce
- **Color mode:** Dark (deep navy)
- **Priority metrics:** Revenue, NPS, SLA, shipped items
- **Special elements:** Production timeline, donut chart for volume, operation availability chart
- **Layout:** Dense, data-forward, minimal decoration

### 8.4 Security Operations
- **Color mode:** Dark (near-black with red/cyan accents)
- **Priority metrics:** Severity counts, risk level, security posture score, incident response
- **Special elements:** World map with threat pins, inside threat bar chart, hexagonal device cards
- **Central element:** Security posture hub (large radial gauge with satellite program scores)

### 8.5 Customer Engagement / Retail
- **Color mode:** Dark (deep navy with warm accents)
- **Priority metrics:** Conversion rates, bounce rates, NPS, session duration, support queue
- **Special elements:** Channel comparison (Mobile/Web/In-Store columns), NPS gain/loss chart
- **Layout:** Column-per-channel with matching metrics for easy comparison

### 8.6 Manufacturing / Production
- **Color mode:** Dark (navy with purple/green accents)
- **Priority metrics:** OEE, availability, performance, quality, energy, production cost
- **Special elements:** Production flow diagram, shift comparison, incident tables per station
- **Layout:** Production flow (left station → center hub → right station) + summary strip

---

## 9. REACT-TO-STUDIO TRANSLATION MAP

### 9.1 Component Mapping
| React Component          | Dashboard Studio Visualization     |
|--------------------------|-------------------------------------|
| KPI Card (number+trend)  | `viz.singlevalue` with options     |
| Bar/Column Chart         | `viz.bar` or `viz.column`          |
| Line Chart / Sparkline   | `viz.line`                         |
| Donut Chart              | `viz.pie` (ring mode)              |
| Radial Gauge             | `viz.radialGauge`                  |
| Table                    | `viz.table`                        |
| Text Block               | `viz.markdown`                     |
| HTML/SVG Custom          | `viz.markdown` with raw HTML       |
| Background Image         | `viz.markdown` at z-index 0        |
| Connecting Lines         | `viz.markdown` with SVG            |
| Geographic Map           | `viz.choropleth` or SVG overlay    |
| Dropdown Filter          | `input.dropdown`                   |
| Time Range Picker        | `input.timerange`                  |

### 9.2 Layout Translation
- React `flexbox` rows → Studio absolute positioning, calculate x/y from flex layout
- React `grid` → Studio absolute positioning, map grid cells to x/y/w/h
- React padding/margin → Studio panel gaps (typically 8-16px between panels)
- React responsive breakpoints → Studio fixed canvas width (typically 1440-1920px)

### 9.3 Styling Translation
- React CSS `background-color` → Studio viz `options.backgroundColor` or canvas background
- React CSS `color` → Studio viz `options.fontColor` or individual option overrides
- React CSS `font-size` → Studio viz `options.majorFontSize`, `options.trendFontSize`, etc.
- React CSS `border-radius` → Not directly supported in most viz; use `viz.markdown` wrapper
- React CSS `box-shadow` → Not supported; use border or background contrast instead

---

## 10. REACT PROTOTYPING WORKFLOW

### 10.1 Phase Sequence
1. **Design Brief** — Define vertical, target audience, key metrics, data sources
2. **Component Selection** — Choose layout architecture (Section 2) and KPI card types (Section 4)
3. **React Prototype** — Build in Claude/Claude Code using Tailwind CSS
4. **Visual Validation** — Review with stakeholders, iterate on colors/layout
5. **Translation Plan** — Map each React component to Studio viz type (Section 9)
6. **Studio JSON Generation** — Generate Dashboard Studio JSON with correct absolute positioning
7. **SPL Integration** — Wire up dataSources with production SPL queries
8. **Testing & Polish** — Validate in Splunk, adjust positioning, test with live data

### 10.2 React Prototype Conventions
- Use Tailwind CSS for rapid styling
- Use `recharts` for chart prototypes (maps well to Studio chart types)
- Hard-code sample data that matches expected SPL output schema
- Use a `DASHBOARD_CONFIG` object at top of file for easy color/layout tweaking
- Comment each section with the intended Studio viz type

### 10.3 Prototype-to-JSON Checklist
- [ ] Every React component has a mapped Studio viz type
- [ ] All absolute positions (x, y, width, height) are calculated from prototype layout
- [ ] Color values are extracted and documented
- [ ] Font sizes are mapped to Studio font size options
- [ ] DataSource stubs are created for each dynamic value
- [ ] Token dependencies are identified (filters → queries → viz)
- [ ] Custom HTML/SVG elements are isolated into viz.markdown panels
- [ ] Background and decorative elements have correct z-ordering

---

## 11. QUALITY CHECKLIST

### Before Handoff
- [ ] All KPI cards follow consistent anatomy (value + unit + trend + delta)
- [ ] Color coding follows the vertical's color system (Section 3)
- [ ] Trend arrows use context-appropriate coloring (↑ cost = red, ↑ uptime = green)
- [ ] Charts have appropriate axis labels and legends
- [ ] Section containers have consistent border radius and padding
- [ ] Header includes logo, title, time picker, and current date/time
- [ ] Background treatment is appropriate for the vertical
- [ ] Mobile/narrow viewport graceful degradation is considered
- [ ] All text is readable against its background (contrast ratio ≥ 4.5:1)
- [ ] Custom HTML/SVG panels are self-contained and don't break on data refresh
