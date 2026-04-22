---
name: splunk-ui-toolkit
description: Comprehensive guide for Splunk UI development including the Splunk UI Toolkit (SUIT), Dashboard Studio, @splunk/react-ui components, visualizations, design principles, and dashboard best practices. Use when Claude needs to help with (1) Creating custom Splunk apps with React, (2) Building dashboards in Dashboard Studio, (3) Using @splunk/react-ui components, (4) Implementing visualizations, (5) Dashboard design and UX best practices, (6) Dynamic options syntax (DOS), (7) Tokens and drilldowns, (8) Theming and accessibility, or (9) Any Splunk frontend development task.
---
 
# Splunk UI Toolkit
 
## Overview
 
The Splunk UI Toolkit (SUIT) is a collection of packages and libraries for building enterprise-grade Splunk applications using React. It provides the same underlying tools that power Splunk's product line.
 
### Core Packages
 
| Package | Purpose |
|---------|---------|
| `@splunk/react-ui` | React component library implementing Splunk design language |
| `@splunk/visualizations` | Chart and visualization components (Highcharts/D3.js based) |
| `@splunk/dashboard-core` | Base dashboard rendering engine |
| `@splunk/dashboard-context` | Dashboard state and context management |
| `@splunk/dashboard-presets` | Pre-configured visualization sets |
| `@splunk/create` | CLI scaffolding tool for new apps |
| `@splunk/themes` | Theming system (dark/light modes) |
| `@splunk/splunk-utils` | Utilities for Splunk API interaction |
 
## Quick Start with @splunk/create
 
```bash
mkdir my-splunk-app && cd my-splunk-app
npx @splunk/create
# Follow prompts: app name, page name, page type
yarn setup
yarn link:app  # Links to local Splunk instance
yarn start     # Start development
```
 
Generated structure:
```
packages/
├── my-splunk-app/     # Splunk app directory
└── my-page/           # React page source
```
 
## Dashboard Development Approaches
 
### 1. Dashboard Studio (JSON-based)
- Built-in UI editor with visual and source modes
- JSON definition wrapped in XML for Splunk
- Best for: Standard dashboards, non-developers, quick iteration
 
### 2. Classic Dashboards (Simple XML)
- XML-based configuration
- Legacy but widely used
- Best for: Compatibility with older Splunk versions
 
### 3. Splunk UI Toolkit (React)
- Full React application development
- Maximum flexibility and customization
- Best for: Complex apps, custom workflows, advanced interactivity
 
## Dashboard Definition Structure (JSON)
 
```json
{
  "title": "Dashboard Title",
  "description": "Dashboard description",
  "inputs": {},
  "defaults": {
    "dataSources": {
      "ds.search": {
        "options": {
          "queryParameters": {
            "earliest": "$global_time.earliest$",
            "latest": "$global_time.latest$"
          }
        }
      }
    }
  },
  "visualizations": {},
  "dataSources": {},
  "layout": {}
}
```
 
### Key Sections
 
**dataSources**: Search definitions
```json
"ds_search1": {
  "type": "ds.search",
  "options": {
    "query": "index=_internal | stats count by sourcetype",
    "queryParameters": {
      "earliest": "-24h",
      "latest": "now"
    },
    "refresh": "10s",
    "refreshType": "delay"
  },
  "name": "Search_1"
}
```
 
**visualizations**: Chart definitions
```json
"viz_pie": {
  "type": "splunk.pie",
  "dataSources": { "primary": "ds_search1" },
  "options": { "labelDisplay": "valuesAndPercentage" },
  "title": "Distribution"
}
```
 
**inputs**: User input controls
```json
"input_timerange": {
  "type": "input.timerange",
  "options": {
    "token": "global_time",
    "defaultValue": "-24h@h,now"
  },
  "title": "Time Range"
}
```
 
**layout**: Positioning (grid or absolute)
```json
"layout": {
  "type": "grid",
  "globalInputs": ["input_timerange"],
  "structure": [
    {
      "item": "viz_pie",
      "type": "block",
      "position": { "x": 0, "y": 0, "w": 600, "h": 400 }
    }
  ]
}
```
 
## Visualization Types
 
### Charts (splunk.* prefix)
- `splunk.line` / `splunk.area` - Time series trends
- `splunk.column` / `splunk.bar` - Categorical comparisons
- `splunk.pie` - Part-to-whole relationships
- `splunk.scatter` / `splunk.bubble` - Correlations
- `splunk.sankey` - Flow/movement between entities
- `splunk.linkgraph` - Relationship networks
 
### Single Values
- `splunk.singlevalue` - Key metrics display
- `splunk.singlevalueicon` - Metric with icon
- `splunk.singlevalueradial` - Radial gauge style
 
### Gauges
- `splunk.fillergauge` - Horizontal fill indicator
- `splunk.markergauge` - Marker on scale
 
### Tables & Maps
- `splunk.table` - Tabular data with formatting
- `splunk.choropleth` - Geographic heatmaps
- `splunk.map` - Marker/bubble maps
 
### Other
- `splunk.markdown` - Text/documentation
- `splunk.image` - Static images
- `splunk.rectangle` / `splunk.ellipse` - Shapes
 
## Dynamic Options Syntax (DOS)
 
DOS enables data-driven visualization styling. Structure:
```
"> [data source] | [selector functions] | [formatting function]"
```
 
### Selector Functions
- `seriesByName('fieldname')` - Select data series
- `lastPoint()` / `firstPoint()` - Get specific data point
- `delta(-N)` - Calculate change from N points back
 
### Formatting Functions
- `matchValue(config)` - Exact value matching
- `rangeValue(config)` - Range-based coloring
- `gradient(config)` - Color gradients
- `formatByType(config)` - Number/string formatting
 
### Example: Dynamic Single Value Color
```json
"options": {
  "majorValue": "> primary | seriesByName('count') | lastPoint()",
  "trendValue": "> primary | seriesByName('count') | delta(-2)",
  "backgroundColor": "> primary | seriesByName('count') | lastPoint() | rangeValue(colorConfig)"
},
"context": {
  "colorConfig": [
    { "value": 0, "color": "#DC4E41" },
    { "value": 50, "color": "#F8BE34" },
    { "value": 100, "color": "#53A051" }
  ]
}
```
 
## Tokens and Drilldowns
 
### Token Syntax
- Reference: `$token_name$`
- Set default in `defaults.tokens`
- Filter tokens: `$token|s$` (safe), `$token|n$` (numeric)
 
### Dashboard Studio Drilldown
```json
"eventHandlers": [
  {
    "type": "drilldown.setToken",
    "options": {
      "tokens": [
        { "token": "selected_host", "key": "row.host.value" }
      ]
    }
  }
]
```
 
### Simple XML Drilldown
```xml
<drilldown>
  <condition match="$click.value$ != &quot;(null)&quot;">
    <set token="selected">$click.value$</set>
    <link target="_blank">/app/search/dashboard?host=$row.host$</link>
  </condition>
</drilldown>
```
 
### Predefined Drilldown Tokens
- `$click.value$` / `$click.value2$` - Clicked values
- `$click.name$` / `$click.name2$` - Field names
- `$row.<fieldname>$` - Full row access
- `$earliest$` / `$latest$` - Time range from click
 
## @splunk/react-ui Components
 
### Installation
```bash
npm install @splunk/react-ui react react-dom styled-components
```
 
### Key Components
- **Layout**: `ColumnLayout`, `ControlGroup`, `SidePanel`
- **Input**: `Button`, `Text`, `Select`, `Multiselect`, `Switch`
- **Display**: `Table`, `Card`, `Accordion`, `Modal`, `Tooltip`
- **Navigation**: `Menu`, `TabBar`, `Breadcrumb`
- **Feedback**: `Message`, `Toast`, `WaitSpinner`
 
### Theming
```jsx
import { SplunkThemeProvider } from '@splunk/themes';
 
<SplunkThemeProvider family="enterprise" colorScheme="dark">
  <App />
</SplunkThemeProvider>
```
 
### Fonts
Default font stacks (require licensing):
- Sans: Splunk Platform Sans, Proxima Nova, Roboto, Helvetica Neue
- Mono: Splunk Platform Mono, Inconsolata, Consolas
 
## Dashboard Design Best Practices
 
### Layout Principles
1. **Hierarchy**: Most important metrics at top/center
2. **Grouping**: Related visualizations together
3. **Flow**: Top-to-bottom, left-to-right scanning
4. **White space**: Avoid clutter, use margins
 
### Dashboard Structure Pattern
```
┌─────────────────────────────────────┐
│ Title + Description + Time Picker   │  ← Context
├─────────────────────────────────────┤
│ KPI 1 │ KPI 2 │ KPI 3 │ KPI 4      │  ← Summary metrics
├─────────────────────────────────────┤
│ Trend Chart (time series)           │  ← Trends
├──────────────────┬──────────────────┤
│ Category View    │ Detail Table     │  ← Breakdown + Detail
└──────────────────┴──────────────────┘
```
 
### Visualization Selection
| Data Type | Best Visualization |
|-----------|-------------------|
| Single metric | Single Value |
| Trend over time | Line Chart |
| Discrete time points | Column Chart |
| Part-to-whole | Pie (≤6 slices) |
| Comparison across categories | Bar Chart |
| Correlation (2 vars) | Scatter Plot |
| Correlation (3 vars) | Bubble Chart |
| Flow/movement | Sankey Diagram |
| Geographic | Choropleth / Map |
| Detailed records | Table |
 
### Color Guidelines
 
**Color Types**:
- **Sequential**: Single hue gradient for continuous data (light to dark)
- **Diverging**: Two hues with neutral midpoint for pos/neg values
- **Categorical**: Distinct hues for discrete categories (max 6-8)
 
**Best Practices**:
- Use neutral gray for most data, highlight with color
- Red/green for good/bad states (consider colorblindness)
- Match saturation levels across categorical colors
- Test in grayscale for accessibility
- Limit to 7 colors max per visualization
 
**Accessibility**:
- 8% of men are red-green colorblind
- Use shape/pattern alongside color when possible
- Ensure 4.5:1 contrast ratio for text
- Follow WCAG 2.2 Level AA guidelines
 
### Performance Optimization
- Use base searches with chain searches for efficiency
- Avoid real-time searches on dashboards
- Limit to 10,000 data points per visualization
- Use `refresh` and `refreshType` wisely
- Consider scheduled reports for heavy searches
 
## Theming
 
### Dashboard Theme (XML wrapper)
```xml
<dashboard version="2" theme="dark">
  <label>My Dashboard</label>
  <definition><![CDATA[{...JSON...}]]></definition>
</dashboard>
```
 
### App Theme Support
In `app.conf`:
```ini
[ui]
supported_themes = light, dark
```
 
### CSS Customization (Simple XML)
```xml
<dashboard stylesheet="custom.css" theme="dark">
```
 
## App Structure
 
```
my-app/
├── default/
│   ├── app.conf
│   ├── data/ui/views/      # Dashboards
│   └── data/ui/nav/        # Navigation
├── appserver/
│   └── static/             # CSS, JS, images
├── bin/                    # Scripts
└── README
```
 
### app.conf Essential Settings
```ini
[install]
is_configured = 0
 
[ui]
is_visible = true
label = My App
supported_themes = light, dark
 
[launcher]
description = App description
version = 1.0.0
```
 
## Common Patterns
 
### Base + Chain Search Pattern
```json
"dataSources": {
  "ds_base": {
    "type": "ds.search",
    "options": { "query": "index=web | stats count by status, host" }
  },
  "ds_by_status": {
    "type": "ds.chain",
    "options": {
      "query": "| stats sum(count) by status",
      "extend": "ds_base"
    }
  },
  "ds_by_host": {
    "type": "ds.chain",
    "options": {
      "query": "| stats sum(count) by host",
      "extend": "ds_base"
    }
  }
}
```
 
### Conditional Panel Visibility
```json
"visualizations": {
  "viz_detail": {
    "type": "splunk.table",
    "options": { "depends": "$selected_host$" }
  }
}
```
 
## Troubleshooting
 
| Issue | Solution |
|-------|----------|
| Blank visualization | Check data format matches viz requirements |
| Token not updating | Verify token name matches exactly, check filters |
| Slow dashboard | Use base searches, reduce time range, add filters |
| CSS not loading | Clear browser cache, check file paths |
| Theme not applying | Verify `supported_themes` in app.conf |
 
## Resources
 
- [Splunk UI Documentation](https://splunkui.splunk.com)
- [Developer Portal](https://dev.splunk.com)
- [Dashboard Studio Docs](https://docs.splunk.com/Documentation/Splunk/latest/DashStudio)
- [Examples Gallery](https://splunkui.splunk.com/Examples)
