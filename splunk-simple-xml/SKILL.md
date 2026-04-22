---
name: splunk-simple-xml
description: >
  Use this skill when the user wants to create, edit, or troubleshoot a Splunk Classic (Simple XML)
  dashboard. Triggers include: building a Simple XML dashboard (.xml), adding rows/panels/charts/
  tables/single-value panels, configuring inline or base searches, setting up inputs (time range,
  dropdown, text, radio, multiselect, checkbox), using tokens, writing drilldown actions, configuring
  panel options (charting.*), auto-refresh, fieldset, post-process searches, or saving dashboards
  inside a Splunk app. Does NOT cover Dashboard Studio (JSON/version="2") — use splunk-dashboards
  skill for that.
---

# Splunk Simple XML (Classic Dashboards) Skill

## What this skill does

Generates correct, working Simple XML dashboard definitions. All output is valid XML that can be
copy-pasted directly into a Splunk app's `default/data/ui/views/` directory. Follows Splunk
Enterprise and Splunk Cloud Platform Simple XML specification.

---

## CRITICAL RULES — always enforce these

1. **Root element** must be `<dashboard>` (not `<form>` unless inputs are needed — see below)
2. **Use `<form>`** as root element when the dashboard has any `<input>` controls
3. **Every `<panel>`** must be inside a `<row>`
4. **Panel content** is a single element: `<chart>`, `<table>`, `<single>`, `<map>`, `<event>`,
   `<html>`, or `<list>`
5. **Searches** live inside the panel element, not the `<panel>` tag itself
6. **`<fieldset>`** must be a direct child of `<form>` (not inside a row)
7. **Token references** use `$token_name$` syntax in queries and option values
8. **Base search IDs** are referenced with `<search base="search_id">` — the base search must have
   `id` attribute set
9. **Never use Dashboard Studio JSON inside a Simple XML file** — they are incompatible formats

---

## Document structure

```xml
<form>                          <!-- Use <dashboard> if no inputs exist -->
  <label>Dashboard Title</label>
  <description>Optional description</description>

  <fieldset submitButton="false" autoRun="true">
    <!-- inputs go here -->
  </fieldset>

  <row>
    <panel>
      <!-- panel content -->
    </panel>
  </row>

</form>
```

**Root element attributes:**

| Attribute | Values | Purpose |
|-----------|--------|---------|
| `theme` | `light`, `dark` | Dashboard color theme |
| `stylesheet` | `custom.css` | Custom CSS file from `appserver/static/` |
| `isVisible` | `true`, `false` | Show/hide in navigation |
| `onunloadCancelJobs` | `true`, `false` | Cancel running searches when navigating away |

---

## Searches

### Inline search (most common)
```xml
<chart>
  <search>
    <query>index=main sourcetype=syslog | timechart count by host</query>
    <earliest>$time_picker.earliest$</earliest>
    <latest>$time_picker.latest$</latest>
    <refresh>60s</refresh>
    <refreshType>delay</refreshType>
  </search>
  <option name="charting.chart">line</option>
</chart>
```

### Base search + post-process (share one search across multiple panels)
```xml
<!-- Base search — define once, reuse many times -->
<search id="base_sensor_data">
  <query>index=ot_sensors site=$selected_site$
| stats latest(temperature) as temp, latest(pressure) as pressure,
         latest(status) as status by equipment_id</query>
  <earliest>$time_picker.earliest$</earliest>
  <latest>$time_picker.latest$</latest>
</search>

<!-- Panel 1: post-process the base search -->
<table>
  <search base="base_sensor_data">
    <query>| table equipment_id, temp, pressure, status</query>
  </search>
</table>

<!-- Panel 2: another post-process -->
<chart>
  <search base="base_sensor_data">
    <query>| stats avg(temp) as avg_temp by equipment_id</query>
  </search>
  <option name="charting.chart">bar</option>
</chart>
```

### Saved search reference
```xml
<chart>
  <search>
    <searchName>my_saved_search_name</searchName>
    <earliest>$time_picker.earliest$</earliest>
    <latest>$time_picker.latest$</latest>
  </search>
</chart>
```

### Search attributes
| Attribute/Element | Example | Purpose |
|-------------------|---------|---------|
| `id` | `id="base_search"` | Identifies base search for post-processing |
| `<refresh>` | `30s`, `5m`, `1h` | Auto-refresh interval |
| `<refreshType>` | `delay`, `interval` | `delay` = wait for finish; `interval` = fixed clock |
| `<earliest>` | `$time_picker.earliest$` | Start time — always use token from time input |
| `<latest>` | `$time_picker.latest$` | End time |
| `<sampleRatio>` | `10` | Sample 1 in N events |

---

## Panel types

### `<chart>` — all chart types
```xml
<row>
  <panel>
    <title>Temperature Over Time</title>
    <chart>
      <search>
        <query>index=ot_sensors metric=temperature
| timechart avg(value) as avg_temp by equipment_id</query>
        <earliest>$time_picker.earliest$</earliest>
        <latest>$time_picker.latest$</latest>
      </search>
      <option name="charting.chart">line</option>
      <option name="charting.chart.nullValueMode">connect</option>
      <option name="charting.legend.placement">bottom</option>
      <option name="charting.axisTitleX.text">Time</option>
      <option name="charting.axisTitleY.text">Temperature (°C)</option>
      <option name="charting.seriesColors">[0x5C9BD6,0xF5A623,0x7ED321,0xD0021B]</option>
      <option name="height">300</option>
    </chart>
  </panel>
</row>
```

**`charting.chart` values:**
`line`, `area`, `column`, `bar`, `scatter`, `bubble`, `pie`, `radialGauge`, `fillerGauge`,
`markerGauge`, `punchcard`, `sankey`, `choropleth`

**Common charting options:**

| Option | Example values | Purpose |
|--------|---------------|---------|
| `charting.chart` | `line` | Chart type |
| `charting.chart.nullValueMode` | `connect`, `gaps`, `zero` | How to handle null/missing values |
| `charting.legend.placement` | `bottom`, `right`, `top`, `none` | Legend position |
| `charting.axisTitleX.text` | `"Time"` | X-axis label |
| `charting.axisTitleY.text` | `"Count"` | Y-axis label |
| `charting.axisTitleY2.text` | `"Rate"` | Secondary Y-axis label |
| `charting.seriesColors` | `[0xFF6B35,0x5C9BD6]` | Series colors (0x hex) |
| `charting.chart.stackMode` | `default`, `stacked`, `stacked100` | Stacking mode |
| `charting.chart.style` | `shiny`, `minimal` | Visual style |
| `charting.data.count` | `100` | Max data points |
| `charting.drilldown` | `all`, `none` | Enable/disable drilldown |
| `height` | `300` | Panel height in pixels |
| `charting.fieldColors` | `{"CRITICAL": 0xFF0000}` | Color specific series by name |

---

### `<table>` — tabular data
```xml
<row>
  <panel>
    <title>Equipment Status</title>
    <table>
      <search>
        <query>index=ot_sensors
| stats latest(temperature) as Temperature, latest(status) as Status by equipment_id
| rename equipment_id as "Equipment ID"</query>
        <earliest>$time_picker.earliest$</earliest>
        <latest>$time_picker.latest$</latest>
      </search>
      <option name="count">20</option>
      <option name="dataOverlayMode">none</option>
      <option name="drilldown">row</option>
      <option name="rowNumbers">false</option>
      <option name="wrap">true</option>
      <format type="color" field="Status">
        <colorPalette type="map">{"CRITICAL":#FF0000,"WARNING":#FF6B35,"OK":#00CC44}</colorPalette>
      </format>
      <format type="color" field="Temperature">
        <colorPalette type="minMidMax" minColor="#00CC44" midColor="#FF6B35" maxColor="#FF0000">
          <scale type="minMidMax" minValue="0" midValue="70" maxValue="100"/>
        </colorPalette>
      </format>
    </table>
  </panel>
</row>
```

**Table options:**

| Option | Values | Purpose |
|--------|--------|---------|
| `count` | integer | Rows per page |
| `drilldown` | `row`, `cell`, `none` | Click behavior |
| `rowNumbers` | `true`, `false` | Show row numbers |
| `wrap` | `true`, `false` | Wrap long cell text |
| `dataOverlayMode` | `none`, `heatmap`, `highlow` | Overlay mode |

---

### `<single>` — single value / KPI
```xml
<row>
  <panel>
    <title>Avg Temperature</title>
    <single>
      <search>
        <query>index=ot_sensors metric=temperature
| stats avg(value) as avg_temp
| eval avg_temp=round(avg_temp, 1)</query>
        <earliest>$time_picker.earliest$</earliest>
        <latest>$time_picker.latest$</latest>
      </search>
      <option name="unit">°C</option>
      <option name="unitPosition">after</option>
      <option name="underLabel">Average Temperature</option>
      <option name="colorMode">block</option>
      <option name="rangeColors">["0x65A637","0xF7BC38","0xF58F39","0xD93F3C"]</option>
      <option name="rangeValues">[60,75,90]</option>
      <option name="useColors">1</option>
      <option name="trendColorInterpretation">standard</option>
    </single>
  </panel>
</row>
```

**Single value options:**

| Option | Values | Purpose |
|--------|--------|---------|
| `unit` | `°C`, `%`, `ms` | Unit string |
| `unitPosition` | `before`, `after` | Unit placement |
| `underLabel` | string | Label below the value |
| `colorMode` | `none`, `block`, `mini` | How color is applied |
| `rangeColors` | `["0x65A637","0xD93F3C"]` | Colors per range (0x hex) |
| `rangeValues` | `[60,80]` | Threshold values between colors |
| `useColors` | `0`, `1` | Enable range coloring |
| `trendDisplayMode` | `absolute`, `percent` | Trend display format |
| `numberPrecision` | `0`, `2` | Decimal places |

---

### `<map>` — cluster map / choropleth
```xml
<!-- Cluster map -->
<map>
  <search>
    <query>index=access_logs | iplocation clientip | geostats count by action</query>
  </search>
  <option name="mapping.type">marker</option>
  <option name="mapping.markerLayer.markerOpacity">0.8</option>
  <option name="mapping.map.zoom">2</option>
  <option name="mapping.map.center">(0,0)</option>
</map>

<!-- Choropleth map -->
<map>
  <search>
    <query>index=sales | stats sum(revenue) as revenue by country</query>
  </search>
  <option name="mapping.type">choropleth</option>
  <option name="mapping.choroplethLayer.colorMode">sequential</option>
  <option name="mapping.choroplethLayer.maximumColor">0x0099e6</option>
  <option name="mapping.choroplethLayer.minimumColor">0xd4e4f5</option>
</map>
```

---

### `<event>` — raw events viewer
```xml
<event>
  <search>
    <query>index=main sourcetype=syslog severity=CRITICAL</query>
    <earliest>$time_picker.earliest$</earliest>
    <latest>$time_picker.latest$</latest>
  </search>
  <option name="count">10</option>
  <option name="type">list</option>
  <option name="displayRowNumbers">false</option>
</event>
```

---

### `<html>` — custom HTML panel
```xml
<html>
  <![CDATA[
    <div style="padding:20px; text-align:center;">
      <h2 style="color:#5C9BD6;">Equipment Health Dashboard</h2>
      <p>Real-time monitoring of OT sensor data from $selected_site$</p>
    </div>
  ]]>
</html>
```

---

## Inputs (controls)

All inputs must be inside a `<fieldset>` tag. The root element must be `<form>` (not `<dashboard>`)
when inputs are present.

### Time range picker
```xml
<fieldset submitButton="false" autoRun="true">
  <input type="time" token="time_picker" searchWhenChanged="true">
    <label>Time Range</label>
    <default>
      <earliest>-24h@h</earliest>
      <latest>now</latest>
    </default>
  </input>
</fieldset>
```

### Dropdown — static list
```xml
<input type="dropdown" token="selected_site" searchWhenChanged="true">
  <label>Site</label>
  <default>*</default>
  <choice value="*">All Sites</choice>
  <choice value="site_a">Site A</choice>
  <choice value="site_b">Site B</choice>
</input>
```

### Dropdown — dynamic (populated from a search)
```xml
<input type="dropdown" token="selected_equipment" searchWhenChanged="true">
  <label>Equipment</label>
  <default>*</default>
  <choice value="*">All Equipment</choice>
  <search>
    <query>index=ot_sensors | stats count by equipment_id | fields equipment_id</query>
    <earliest>-24h</earliest>
    <latest>now</latest>
  </search>
  <fieldForLabel>equipment_id</fieldForLabel>
  <fieldForValue>equipment_id</fieldForValue>
</input>
```

### Text input
```xml
<input type="text" token="search_filter" searchWhenChanged="false">
  <label>Filter</label>
  <default>*</default>
  <prefix>equipment_id=</prefix>
  <suffix></suffix>
</input>
```

### Radio buttons
```xml
<input type="radio" token="view_mode" searchWhenChanged="true">
  <label>View</label>
  <default>summary</default>
  <choice value="summary">Summary</choice>
  <choice value="detail">Detail</choice>
</input>
```

### Multiselect / Checkbox
```xml
<input type="multiselect" token="selected_sites" searchWhenChanged="true">
  <label>Sites</label>
  <default>*</default>
  <choice value="*">All</choice>
  <choice value="site_a">Site A</choice>
  <choice value="site_b">Site B</choice>
  <delimiter> OR </delimiter>
  <prefix>site IN (</prefix>
  <suffix>)</suffix>
</input>
```

### Checkbox (single toggle)
```xml
<input type="checkbox" token="show_alerts" searchWhenChanged="true">
  <label>Show Alerts Only</label>
  <choice value="AND severity=CRITICAL">Alerts Only</choice>
  <default></default>
</input>
```

**Input attributes:**

| Attribute | Purpose |
|-----------|---------|
| `token` | Token name — reference as `$token_name$` in queries |
| `searchWhenChanged` | `true` = triggers re-run on every change; `false` = only on submit |
| `<prefix>` | Text prepended to token value in queries |
| `<suffix>` | Text appended to token value in queries |
| `<default>` | Initial/default value |

---

## Token usage in queries

Tokens are referenced as `$token_name$` anywhere in a query string.

```xml
<!-- Search using tokens -->
<query>index=ot_sensors site=$selected_site$ equipment_id=$selected_equipment$
| timechart avg(temperature) as temp</query>
```

**Token pipeline — set a secondary token from an input:**
```xml
<input type="dropdown" token="selected_site_label" searchWhenChanged="true">
  <label>Site</label>
  <fieldForLabel>site_name</fieldForLabel>
  <fieldForValue>site_id</fieldForValue>
  <!-- fieldForLabel sets a separate $selected_site_label.label$ token -->
</input>
```

**Eval-based token transformation:**
Use `<condition>` inside inputs to set or rewrite tokens:
```xml
<input type="dropdown" token="status_filter" searchWhenChanged="true">
  <label>Status</label>
  <choice value="all">All</choice>
  <choice value="critical">Critical</choice>
  <change>
    <condition value="all">
      <set token="status_query"></set>
    </condition>
    <condition value="critical">
      <set token="status_query">AND status="CRITICAL"</set>
    </condition>
  </change>
</input>
```

---

## Drilldowns

### Set a token from a table row click (most common pattern)
```xml
<table>
  <search>...</search>
  <option name="drilldown">row</option>
  <drilldown>
    <set token="selected_equipment">$row.equipment_id$</set>
    <set token="drilldown_visible">true</set>
  </drilldown>
</table>
```

### Navigate to another dashboard on click
```xml
<chart>
  <search>...</search>
  <option name="charting.drilldown">all</option>
  <drilldown>
    <link target="_blank">/app/my_app/equipment_detail?equipment_id=$click.value$</link>
  </drilldown>
</chart>
```

### Conditional drilldown (different action per value)
```xml
<table>
  <option name="drilldown">cell</option>
  <drilldown>
    <condition field="status">
      <set token="status_selected">$click.value$</set>
    </condition>
    <condition>
      <!-- default: all other clicks -->
      <link>/app/my_app/detail?id=$row.equipment_id$</link>
    </condition>
  </drilldown>
</table>
```

**Drilldown token references:**

| Token | Source |
|-------|--------|
| `$click.value$` | Value of clicked cell or chart element |
| `$click.name$` | Column/series name of clicked element |
| `$row.<fieldname>$` | Any field value from clicked table row |
| `$earliest$` / `$latest$` | Current time range |

### Unset a token (hide a panel)
```xml
<drilldown>
  <unset token="selected_equipment"/>
</drilldown>
```

---

## Conditional panel visibility

Use `<panel depends="$token$">` to show/hide panels based on token state:

```xml
<!-- This panel only shows when $selected_equipment$ token is set -->
<row>
  <panel depends="$selected_equipment$">
    <title>Detail for $selected_equipment$</title>
    <table>
      <search>
        <query>index=ot_sensors equipment_id=$selected_equipment$
| table _time, metric, value</query>
      </search>
    </table>
  </panel>
</row>

<!-- This panel only shows when token is NOT set -->
<row>
  <panel rejects="$selected_equipment$">
    <html><![CDATA[<p>Click a row above to see equipment detail.</p>]]></html>
  </panel>
</row>
```

---

## Panel layout — rows and columns

Panels in the same `<row>` are displayed side by side. Use `<panel id>` for panel grouping.

```xml
<!-- Two panels side by side -->
<row>
  <panel>
    <title>Panel Left</title>
    <single>...</single>
  </panel>
  <panel>
    <title>Panel Right</title>
    <chart>...</chart>
  </panel>
</row>

<!-- Panel spanning full width -->
<row>
  <panel>
    <title>Full Width Chart</title>
    <chart>...</chart>
  </panel>
</row>

<!-- Three KPIs in a row -->
<row>
  <panel>
    <single>...</single>
  </panel>
  <panel>
    <single>...</single>
  </panel>
  <panel>
    <single>...</single>
  </panel>
</row>
```

> Panels in a row are given equal width by default. Splunk does not support custom widths in
> Simple XML without CSS overrides.

---

## Auto-refresh

Set refresh on individual searches:
```xml
<search>
  <query>index=ot_sensors | stats latest(temperature) as temp</query>
  <refresh>30s</refresh>
  <refreshType>delay</refreshType>
</search>
```

Set refresh on entire form at the root element level:
```xml
<form refresh="60s">
```

---

## Saving in a Splunk app

Save as `<app>/default/data/ui/views/<dashboard_name>.xml`

File name rules:
- Lowercase, underscores only (no spaces, no hyphens)
- e.g., `equipment_health.xml` → visible at `/app/<appname>/equipment_health`

Navigation is configured in `default/data/ui/nav/default.xml`:
```xml
<nav search_view="search" color="#333333">
  <view name="equipment_health" default="true"/>
  <collection label="Monitoring">
    <view name="sensor_trends"/>
    <view name="alerts"/>
  </collection>
</nav>
```

---

## Complete working example

```xml
<form theme="dark">
  <label>Equipment Health Monitor</label>
  <description>Real-time OT sensor monitoring</description>

  <fieldset submitButton="false" autoRun="true">
    <input type="time" token="time_picker" searchWhenChanged="true">
      <label>Time Range</label>
      <default>
        <earliest>-1h@h</earliest>
        <latest>now</latest>
      </default>
    </input>
    <input type="dropdown" token="selected_site" searchWhenChanged="true">
      <label>Site</label>
      <default>*</default>
      <choice value="*">All Sites</choice>
      <choice value="site_a">Site A</choice>
      <choice value="site_b">Site B</choice>
    </input>
  </fieldset>

  <!-- Base search shared by all panels -->
  <search id="base_sensors">
    <query>index=ot_sensors site=$selected_site$
| stats latest(temperature) as Temperature,
         latest(pressure) as Pressure,
         latest(status) as Status
  by equipment_id</query>
    <earliest>$time_picker.earliest$</earliest>
    <latest>$time_picker.latest$</latest>
    <refresh>60s</refresh>
    <refreshType>delay</refreshType>
  </search>

  <!-- KPI row -->
  <row>
    <panel>
      <title>Avg Temperature</title>
      <single>
        <search base="base_sensors">
          <query>| stats avg(Temperature) as value | eval value=round(value,1)</query>
        </search>
        <option name="unit">°C</option>
        <option name="unitPosition">after</option>
        <option name="colorMode">block</option>
        <option name="rangeColors">["0x65A637","0xF7BC38","0xF58F39","0xD93F3C"]</option>
        <option name="rangeValues">[60,75,90]</option>
        <option name="useColors">1</option>
      </single>
    </panel>
    <panel>
      <title>Avg Pressure</title>
      <single>
        <search base="base_sensors">
          <query>| stats avg(Pressure) as value | eval value=round(value,2)</query>
        </search>
        <option name="unit">bar</option>
        <option name="unitPosition">after</option>
      </single>
    </panel>
    <panel>
      <title>Critical Alerts</title>
      <single>
        <search base="base_sensors">
          <query>| where Status="CRITICAL" | stats count</query>
        </search>
        <option name="colorMode">block</option>
        <option name="rangeColors">["0x65A637","0xD93F3C"]</option>
        <option name="rangeValues">[1]</option>
        <option name="useColors">1</option>
      </single>
    </panel>
  </row>

  <!-- Trend chart -->
  <row>
    <panel>
      <title>Temperature Trend</title>
      <chart>
        <search>
          <query>index=ot_sensors site=$selected_site$ metric=temperature
| timechart avg(value) as avg_temp by equipment_id</query>
          <earliest>$time_picker.earliest$</earliest>
          <latest>$time_picker.latest$</latest>
          <refresh>60s</refresh>
          <refreshType>delay</refreshType>
        </search>
        <option name="charting.chart">line</option>
        <option name="charting.chart.nullValueMode">connect</option>
        <option name="charting.legend.placement">bottom</option>
        <option name="charting.axisTitleX.text">Time</option>
        <option name="charting.axisTitleY.text">Temperature (°C)</option>
        <option name="height">300</option>
      </chart>
    </panel>
  </row>

  <!-- Equipment table with drilldown -->
  <row>
    <panel>
      <title>Equipment Status</title>
      <table>
        <search base="base_sensors">
          <query>| rename equipment_id as "Equipment ID"
| table "Equipment ID", Temperature, Pressure, Status</query>
        </search>
        <option name="count">20</option>
        <option name="drilldown">row</option>
        <option name="wrap">false</option>
        <format type="color" field="Status">
          <colorPalette type="map">{"CRITICAL":"#D93F3C","WARNING":"#F58F39","OK":"#65A637"}</colorPalette>
        </format>
        <drilldown>
          <set token="selected_equipment">$row.Equipment ID$</set>
        </drilldown>
      </table>
    </panel>
  </row>

  <!-- Detail panel — only visible after row click -->
  <row>
    <panel depends="$selected_equipment$">
      <title>Detail: $selected_equipment$</title>
      <chart>
        <search>
          <query>index=ot_sensors equipment_id="$selected_equipment$"
| timechart avg(temperature) as temp, avg(pressure) as pressure</query>
          <earliest>$time_picker.earliest$</earliest>
          <latest>$time_picker.latest$</latest>
        </search>
        <option name="charting.chart">line</option>
        <option name="charting.legend.placement">bottom</option>
        <option name="height">250</option>
      </chart>
    </panel>
  </row>

</form>
```

---

## Common mistakes and fixes

| Mistake | Fix |
|---------|-----|
| Using `<dashboard>` root with `<fieldset>` / inputs | Change root to `<form>` |
| `<panel>` directly inside `<form>` or `<dashboard>` | Wrap in `<row>` first |
| Token not resolving (shows as `$token$` literally) | Check spelling; inputs must be inside `<fieldset>` |
| Chart shows no data | Verify SPL in search bar first; check `<earliest>`/`<latest>` are set |
| `charting.seriesColors` not working | Use `0xRRGGBB` format (hex with `0x` prefix), not `#RRGGBB` |
| Base search not picked up | Ensure `<search id="...">` and `<search base="...">` IDs match exactly |
| Dashboard not showing in nav | Check file name in `default/data/ui/nav/default.xml` |
| `<format>` color map not working on table | Wrap color values in quotes inside the JSON map |
| Drilldown token not setting | Ensure `<option name="drilldown">row</option>` is set on the table |

---

## References
- Simple XML dashboard overview: https://docs.splunk.com/Documentation/Splunk/latest/Viz/PanelreferenceforSimplifiedXML
- Tokens in Simple XML: https://docs.splunk.com/Documentation/Splunk/latest/Viz/tokens
- Drilldown reference: https://docs.splunk.com/Documentation/Splunk/latest/Viz/DrilldownIntro
- Color formatting: https://docs.splunk.com/Documentation/Splunk/latest/Viz/Coloringdata
