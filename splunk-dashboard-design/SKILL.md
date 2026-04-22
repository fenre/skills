---
name: splunk-dashboard-design
description: >
  Use this skill when the user wants guidance on making a Splunk dashboard look better, more
  professional, or more intuitive. Triggers include: "make the dashboard look good", "improve
  the layout", "what chart type should I use", "design a monitoring dashboard", "how do I
  organise the panels", "make it visually appealing", choosing color palettes, structuring KPIs,
  grouping panels visually, building an executive summary layout, operational monitoring layout,
  or security investigation layout. Applies to BOTH Simple XML and Dashboard Studio dashboards.
  Use alongside the technical skills (splunk-simple-xml or splunk-dashboards) to produce dashboards
  that are both correct AND visually compelling.
---

# Splunk Dashboard Design Skill

## What this skill does

Provides layout templates, color palettes, visual hierarchy principles, and design patterns for
building professional Splunk dashboards. Works for both Simple XML and Dashboard Studio formats.
Focus is on making dashboards readable, actionable, and visually appealing — not just technically
correct.

---

## Core design principles

### 1. Visual hierarchy: overview → detail
Users scan dashboards top to bottom. Respect this:

```
TOP ROW      → KPIs / single values — the "headline numbers"
SECOND ROW   → Trend charts — "how did we get here?"
LOWER ROWS   → Tables, drill-down detail — "what exactly happened?"
FOOTER       → Navigation links, reference info
```

Never bury the most important number in the middle of the page. If something is worth monitoring,
make it the first thing the eye lands on.

### 2. One purpose per dashboard
A dashboard should answer one question well. Resist adding "just one more chart."

| Dashboard type | One question it answers |
|----------------|------------------------|
| Executive summary | "Is everything healthy right now?" |
| Operational monitoring | "What is happening in real time?" |
| Investigation | "What caused this incident?" |
| Capacity/trend | "How are we trending vs. target?" |

Use tabs or drilldown links to separate concerns rather than cramming everything onto one page.

### 3. Establish a visual grid
Pick a canvas size and stick to it. Then define zones:

**Recommended canvas: 1440 × 960 px (absolute layout)**

```
x=0   x=20  x=720         x=1420
                                   y=20   ← Input bar (h=40)
[           Inputs               ]
                                   y=80   ← KPI zone (h=160)
[ KPI 1 ] [ KPI 2 ] [ KPI 3 ] [ KPI 4 ]
                                   y=260  ← Primary charts (h=340)
[   Main Chart         ] [ Table  ]
                                   y=620  ← Secondary content (h=300)
[   Detail / Secondary Chart       ]
                                   y=940  ← Footer / nav
```

Maintain consistent margins: **20 px** from canvas edges, **20 px** gutters between panels.

---

## Color palettes

### Dark theme — recommended for operational/OT dashboards

**Canvas background:** `#1A1C20` or `#0F1117`
**Panel background:** `#1E2128` or `#252830`
**Card/border:** `#2D3040` or `#3D4456`
**Primary text:** `#FFFFFF` or `#E8EAF0`
**Secondary text:** `#8B90A0` or `#A0A8B8`

**Status colors (dark theme):**
```
Good / OK       #65A637   (green)
Warning         #F5A623   (amber)
High / Alert    #F58F39   (orange)
Critical        #D93F3C   (red)
Info / Neutral  #5C9BD6   (blue)
```

**Data series colors (dark theme) — use in this order:**
```
Series 1   #5C9BD6   Blue
Series 2   #F5A623   Amber
Series 3   #65A637   Green
Series 4   #D0021B   Red
Series 5   #9B59B6   Purple
Series 6   #1FBAD6   Teal
Series 7   #E85E2B   Orange
Series 8   #BC99C7   Lavender
```

### Light theme — recommended for executive / report-style dashboards

**Canvas background:** `#F2F4F7` or `#FFFFFF`
**Panel background:** `#FFFFFF`
**Card/border:** `#E1E4EA`
**Primary text:** `#1D1F25`
**Secondary text:** `#5A6070`

**Status colors (light theme):**
```
Good / OK       #2B9E44
Warning         #D4820A
High / Alert    #C05C00
Critical        #C0392B
Info / Neutral  #2066C0
```

**Data series colors (light theme):**
```
Series 1   #2066C0   Blue
Series 2   #D4820A   Amber
Series 3   #2B9E44   Green
Series 4   #C0392B   Red
Series 5   #7C3AED   Purple
Series 6   #0891B2   Teal
```

### Color rules
- **Status always uses a consistent color system** — green/amber/red everywhere on the dashboard,
  never different colors for OK/WARNING/CRITICAL on different panels
- **Series colors should be consistent** — if equipment_a is blue in chart 1, it must be blue in
  chart 2. Use `seriesColors` with explicit field colors to enforce this.
- **Never use color as the only signal** — always pair color with a label (e.g., "CRITICAL" text
  and red color, not just red)
- **Limit series to 6–8** per chart before splitting into multiple charts
- **Use a single accent color** for interactive/highlighted elements throughout the dashboard

---

## KPI panel design

Single value panels are the most-read elements on a dashboard. Design them carefully.

### KPI sizing guidelines
- **Minimum width:** 200 px in absolute layout
- **Recommended width:** 300–440 px (for a 4-KPI row on 1440px canvas)
- **Height:** 120–160 px
- **Font size:** 36–56 px for the major value

### KPI anatomy
```
┌─────────────────────────┐
│  Panel title            │  ← Small, secondary text
│                         │
│      78.4 °C            │  ← Major value: large, colored
│                         │
│  ↑ 3.2% vs last hour    │  ← Trend indicator (optional)
└─────────────────────────┘
```

### KPI best practices
- **Always use `colorMode: "block"` (Simple XML) or dynamic `majorColor`** to make status
  immediately visible — don't make users read the number to know if it's good or bad
- **Group related KPIs** in a row — availability, performance, quality metrics together
- **Use a background rectangle** (Dashboard Studio) to visually group KPIs as a zone
- **Show a trend** when possible (vs previous period, vs target)
- **Keep units visible** — never make the user wonder if a value is in °C or °F
- **Avoid more than 5–6 KPIs per row** — beyond that, reduce font size or split rows

### KPI grouping with background cards (Dashboard Studio)
Use `splunk.rectangle` shapes positioned slightly behind and slightly larger than the KPI group:

```
Background rectangle:  x=20, y=80, w=1400, h=160
KPI 1:                 x=30, y=90, w=320,  h=140
KPI 2:                 x=370, y=90, w=320, h=140
KPI 3:                 x=710, y=90, w=320, h=140
KPI 4:                 x=1050, y=90, w=350, h=140
```

In `layout.structure`, place rectangle entries **before** the KPIs so they render behind them.

---

## Chart type selection guide

| Question | Best chart type |
|----------|----------------|
| How does a metric change over time? | `line` chart |
| How does a metric change over time with stacked breakdown? | `area` chart (stacked) |
| How do categories compare? | `column` chart (vertical bars) |
| How do many categories compare? | `bar` chart (horizontal bars) |
| How does one number relate to another? | `scatter` plot |
| What is the part-to-whole breakdown? | `pie` chart (max 5 slices) |
| What is the single most important number right now? | `singlevalue` KPI |
| How far is a metric from a target/limit? | `singlevalueradial` or `fillerGauge` |
| What are all the individual events? | `table` |
| Where in the world did events occur? | `map` |
| What is the flow between categories? | `sankey` |

**When NOT to use pie charts:** More than 5 categories, values are similar in size, trend over
time matters. Use a column chart instead.

**When to use tables:** When users need exact values, want to click individual rows, or are comparing
many fields. Tables should never be the only visualization — always pair with a chart.

---

## Layout templates

### Template 1: Operational Monitoring (dark theme)
```
Zones (1440 × 960 canvas):

[20,20] Inputs: time range (w=350), filters (w=250 each)          h=40

[20,80] KPI Zone (background rectangle w=1400 h=160)              h=160
        4 KPI panels side by side (w≈340 each with 10px gaps)

[20,260] Main charts zone                                          h=340
         Primary time series chart (w=860)
         Status table with drilldown (w=520)

[20,620] Secondary zone                                            h=300
         Alert history chart or secondary trend

Margins: 20px edge, 20px gutter between panels
```

### Template 2: Executive Summary (light or dark)
```
[20,20]  Title markdown panel (full width, w=1400, h=50)
[20,80]  Time range input                                          h=40

[20,140] 3–4 headline KPIs (full width)                           h=160

[20,320] Two charts side by side                                   h=300
         Chart 1 (w=680): Primary trend
         Chart 2 (w=680): Secondary trend or breakdown

[20,640] Summary table (full width, w=1400, h=280)
         Shows top items, ranked, with status coloring
```

### Template 3: Investigation / Drill-down (dark theme)
```
[20,20]  Inputs: time range + multiple filter dropdowns            h=40

[20,80]  Overview KPIs row                                         h=120

[20,220] Timeline / event density chart (full width, w=1400)       h=200

[20,440] Split view:                                               h=460
         Left: Filtered events table (w=860, drilldown enabled)
         Right: Detail panel — only visible after row click (w=520)
```

---

## Visual grouping techniques

### Dashboard Studio: using rectangles as cards
Group related panels visually by placing a slightly larger `splunk.rectangle` behind them.

```json
"viz_card_sensors": {
  "type": "splunk.rectangle",
  "options": { "fillColor": "#1E2128", "strokeColor": "#3D4456", "strokeWidth": 1, "rx": 8 }
}
```

Place in `layout.structure` **before** the panels it groups:
```json
{ "item": "viz_card_sensors", "type": "viz", "position": { "x": 20,  "y": 260, "w": 860, "h": 360 } },
{ "item": "viz_temp_chart",   "type": "viz", "position": { "x": 30,  "y": 270, "w": 840, "h": 340 } }
```

10 px margin between card edge and panel content (card x+10 → panel x, card y+10 → panel y,
card w-20 → panel w).

### Dashboard Studio: section title headers
Use `splunk.markdown` for section headers:
```json
"viz_section_title": {
  "type": "splunk.markdown",
  "options": {
    "markdown": "### Sensor Performance",
    "fontColor": "#8B90A0",
    "backgroundColor": "transparent"
  }
}
```

Position above the panels in that section, height 30–40 px.

### Simple XML: visual grouping with `<html>` panels
Use full-width `<html>` panels as section dividers:
```xml
<row>
  <panel>
    <html>
      <![CDATA[
        <div style="padding:8px 0; border-bottom:1px solid #3D4456;">
          <span style="color:#8B90A0; font-size:12px; font-weight:600; 
                       text-transform:uppercase; letter-spacing:1px;">
            Sensor Performance
          </span>
        </div>
      ]]>
    </html>
  </panel>
</row>
```

---

## Input bar design

### Placement and sizing
- Inputs always go at the top of the dashboard (y=20 in absolute layout)
- Time range picker: 300–350 px wide — always first
- Filter dropdowns: 200–250 px wide each
- Keep total input bar height to 40 px (single row)
- Leave right margin space (don't fill all 1440 px with inputs)

### Input naming
Use clear, user-facing labels (not internal field names):
- `"Site"` not `"site_id"`
- `"Time Range"` not `"time_picker"`
- `"Equipment"` not `"equipment_filter_token"`

### Cascade filtering
When inputs are interdependent, order them left to right by scope:
```
[Time Range]  [Region]  [Site]  [Equipment]
   broadest  ←────────────────→  narrowest
```

Dynamic dropdowns should filter based on upstream selections — e.g., Equipment dropdown should
only show equipment that belongs to the selected Site.

---

## Whitespace and spacing

**Rule: crowded dashboards are unread dashboards.**

- Minimum gutter between panels: **20 px**
- Minimum margin from canvas edge: **20 px**
- Minimum panel height for charts: **240 px** (below this, axes become unreadable)
- Single value panel minimum height: **120 px**
- Table minimum height: **200 px** (3–4 visible rows + header)

**When to use tabs instead of more rows:**
- Dashboard has more than 6–8 panels
- Content divides naturally into "Overview" / "Detail" / "Trends" categories
- Some panels are only relevant after a specific action (drilldown)

---

## Typography in Splunk dashboards

### Panel titles
Keep panel titles short (3–5 words): `"Temperature Trend"` not `"Chart showing the temperature 
trend for all monitored equipment over time"`

Use title case for panel titles, sentence case for descriptions.

### Markdown panels for context
Use `splunk.markdown` (Dashboard Studio) or `<html>` panels (Simple XML) for:
- Dashboard header with title, owner, refresh interval
- Section separators with short explanatory text
- "How to use this dashboard" notes

Example header markdown:
```markdown
## Equipment Health Monitor
Real-time sensor monitoring · Auto-refreshes every 60s · Data from index=ot_sensors
```

### Avoid putting too much text on dashboards
Dashboards are read at a glance. If you're writing paragraphs, it belongs in a report, not a
dashboard. Use tooltips (table column descriptions) or drilldown links to detailed documentation.

---

## Conditional coloring — status patterns

Use consistent, unambiguous coloring for status/health indicators across the entire dashboard.

### Recommended status color mapping

| Status | Dark theme | Light theme | Use for |
|--------|-----------|-------------|---------|
| OK / Normal / Good | `#65A637` | `#2B9E44` | All green metrics |
| Warning / Caution | `#F5A623` | `#D4820A` | Metrics approaching limits |
| High / Elevated | `#F58F39` | `#C05C00` | Metrics exceeding soft limit |
| Critical / Fault | `#D93F3C` | `#C0392B` | Metrics at alarm threshold |
| Unknown / No data | `#8B90A0` | `#9B99A0` | Missing or unavailable data |
| Info / Informational | `#5C9BD6` | `#2066C0` | Neutral informational |

### Apply status colors consistently
- Same status → same color on every panel of the dashboard
- If a table row is green for `status=OK`, the KPI for the same equipment must also be green
- Define your threshold values once and reuse them in both chart ranges and table column formats

### OT-specific health thresholds (adjust to your application)
```
Temperature:   0–60°C = OK,  60–80°C = Warning,  80–90°C = High,  >90°C = Critical
Pressure:      0–5 bar = OK, 5–8 bar = Warning,   >8 bar = Critical
Vibration:     ISO 20816 Class I: <1.12 = A, 1.12–2.8 = B, 2.8–7.1 = C, >7.1 = D
Availability:  >98% = OK,  95–98% = Warning,  <95% = Critical
```

---

## Multi-dashboard navigation patterns

### Tab pattern (single file, related content)
Use tabs when: content is closely related, shares the same inputs/time range, and users switch
frequently between views.

```
[Overview] [Site A] [Site B] [Alerts]
```

Tabs in Dashboard Studio → use `layout.tabs`.
Tabs in Simple XML → not natively supported; use separate dashboards with a nav bar.

### Drilldown pattern (master → detail)
Use drilldown when: the detail view is only relevant after selecting an item in the master view,
or the detail requires different filters than the overview.

```
Equipment Summary Dashboard
  → click a row →
    Equipment Detail Dashboard (pre-filtered to selected equipment_id)
```

Pass context via URL tokens (Dashboard Studio `drilldown.linkToDashboard` with tokens, or
Simple XML `<link>` with query params).

### Navigation bar pattern (Simple XML)
Use a full-width `<html>` panel with navigation links as the first row:
```xml
<row>
  <panel>
    <html>
      <![CDATA[
        <div style="background:#1E2128; padding:10px 20px; display:flex; gap:20px;">
          <a href="/app/my_app/overview"
             style="color:#5C9BD6; text-decoration:none; font-size:13px;">Overview</a>
          <a href="/app/my_app/sensors"
             style="color:#8B90A0; text-decoration:none; font-size:13px;">Sensors</a>
          <a href="/app/my_app/alerts"
             style="color:#8B90A0; text-decoration:none; font-size:13px;">Alerts</a>
        </div>
      ]]>
    </html>
  </panel>
</row>
```

---

## Common design mistakes to avoid

| Mistake | Better approach |
|---------|----------------|
| Putting 10+ panels on one dashboard | Use tabs or separate dashboards with drilldown |
| No color-coding of status | Add threshold-based coloring to all KPIs and status fields |
| All panels the same size | Vary sizes to reflect importance — primary chart bigger than secondary |
| No input bar / time picker | Always include a global time range input |
| Raw field names as panel titles | Write human-readable titles and axis labels |
| More than 8 series on one chart | Aggregate to top N + "Other" or split into multiple charts |
| Pie chart with 8+ slices | Use column chart instead |
| Tables without row drilldown | Add drilldown to tables — every row should do something |
| Hardcoded time ranges in searches | Always use time tokens from a time range input |
| Colors that don't mean anything | Reserve green/amber/red strictly for status |
| No whitespace between panels | Minimum 20px gutters — whitespace aids scanning |
| Dashboard auto-refreshes at 5s | Use 30s–60s minimum; 5s hurts performance and readability |

---

## Design checklist before publishing

- [ ] Does the most important information appear in the top 200px of the dashboard?
- [ ] Do all status indicators use consistent green/amber/red coloring?
- [ ] Are all panel titles short and human-readable (not raw field names)?
- [ ] Is there a time range input connected to all searches?
- [ ] Are all panels ≥ 20px apart (no overlapping or touching edges)?
- [ ] Are series colors consistent when the same field appears in multiple charts?
- [ ] Does the dashboard load in under 10 seconds at the default time range?
- [ ] Do all tables have drilldown configured?
- [ ] Are there no more than 6–8 panels before the user needs to scroll?
- [ ] Does the dashboard have a clear title and description?
