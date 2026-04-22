---
name: eds-layouts
description: >-
  Higher-level page layout patterns and composite UI compositions for Equinor
  Design System (EDS) styled applications. Covers page shells (top bar, side
  nav), dashboard layouts (KPI rows, grids), data views (toolbar + table +
  pagination), chat interfaces (messages, input, prompt chips), form layouts,
  detail/settings pages, empty states, and responsive utilities. Use when
  building a dashboard, data view, chat interface, form, landing page, or
  complete page layout with EDS components. Requires eds-foundation and
  eds-components skills. Also use when the user asks for "page template" or
  "layout" in Equinor styling, is building a Databricks App with navigation
  and content areas, or asks to use Equinor colors, Equinor color scheme,
  EDS colors, or EDS theming.
---

# Equinor Design System — Layouts & Patterns

Higher-level page layout patterns and composite UI compositions for EDS-styled
applications. These patterns combine foundation tokens and component patterns into
complete page structures: dashboards, data views, chat interfaces, forms, and
landing pages.

## Prerequisites

This skill requires both **eds-foundation** and **eds-components** skills to be
loaded. All CSS variables and component classes referenced here are defined in
those skills.

## When to apply

- User asks for a "dashboard", "data view", "chat interface", "form", or "landing page"
- User needs a complete page layout with multiple EDS components working together
- User is building a Databricks App with navigation, content areas, and data displays
- User asks for "page template" or "layout" in Equinor styling
- Always load eds-foundation and eds-components first, then this skill

---

## 1. Page Shells

### 1.1 Standard App Shell (Top Bar + Content)

```html
<body>
  <header class="eds-topbar">
    <img src="https://cdn.eds.equinor.com/logo/equinor-logo-horizontal.svg#white"
         alt="Equinor" class="eds-topbar-logo" />
    <span class="eds-topbar-title">App Name</span>
    <div class="eds-topbar-spacer"></div>
    <nav class="eds-topbar-nav">
      <a href="#" class="eds-topbar-link active">Dashboard</a>
      <a href="#" class="eds-topbar-link">Settings</a>
    </nav>
  </header>
  <main class="eds-page-content">
    <!-- Content here -->
  </main>
</body>
```

```css
.eds-page-content {
  max-width: 1200px;
  margin: 0 auto;
  padding: 32px 24px;
}
```

### 1.2 App Shell with Side Navigation

```html
<body>
  <header class="eds-topbar">
    <img src="https://cdn.eds.equinor.com/logo/equinor-logo-horizontal.svg#white"
         alt="Equinor" class="eds-topbar-logo" />
    <span class="eds-topbar-title">App Name</span>
  </header>
  <div class="eds-shell-with-sidenav">
    <aside class="eds-sidenav">
      <div class="eds-sidenav-section">Main</div>
      <a href="#" class="eds-sidenav-item active">Dashboard</a>
      <a href="#" class="eds-sidenav-item">Reports</a>
      <div class="eds-sidenav-section">Settings</div>
      <a href="#" class="eds-sidenav-item">Configuration</a>
    </aside>
    <main class="eds-main-content">
      <!-- Content here -->
    </main>
  </div>
</body>
```

```css
.eds-shell-with-sidenav {
  display: flex;
  height: calc(100vh - 64px);
}

.eds-main-content {
  flex: 1;
  overflow-y: auto;
  padding: 32px 24px;
}

@media (max-width: 768px) {
  .eds-shell-with-sidenav { flex-direction: column; }
  .eds-sidenav {
    width: 100%;
    height: auto;
    border-right: none;
    border-bottom: 1px solid var(--eds-border);
    padding: 8px 0;
  }
}
```

### 1.3 Full-Width Shell (No Max Width)

For data-heavy applications where tables and charts need maximum horizontal space:

```css
.eds-page-content-full {
  padding: 24px;
  height: calc(100vh - 64px);
  overflow-y: auto;
}
```

---

## 2. Dashboard Layout

### 2.1 KPI Row + Grid

```html
<div class="eds-page-content">
  <h1>Dashboard</h1>
  <div class="eds-kpi-row">
    <div class="eds-kpi-card">
      <span class="eds-kpi-label">Total Revenue</span>
      <span class="eds-kpi-value">$24.5M</span>
      <span class="eds-kpi-trend eds-kpi-trend-up">+12.4%</span>
    </div>
    <div class="eds-kpi-card">
      <span class="eds-kpi-label">Active Users</span>
      <span class="eds-kpi-value">1,247</span>
      <span class="eds-kpi-trend eds-kpi-trend-down">-3.1%</span>
    </div>
    <div class="eds-kpi-card">
      <span class="eds-kpi-label">Processing Time</span>
      <span class="eds-kpi-value">2.4s</span>
      <span class="eds-kpi-trend eds-kpi-trend-neutral">0%</span>
    </div>
  </div>
  <div class="eds-grid-2col">
    <div class="eds-card">
      <div class="eds-card-header"><span class="eds-card-title">Chart Area</span></div>
      <div class="eds-card-body"><!-- Chart component --></div>
    </div>
    <div class="eds-card">
      <div class="eds-card-header"><span class="eds-card-title">Recent Activity</span></div>
      <div class="eds-card-body"><!-- Activity list --></div>
    </div>
  </div>
</div>
```

```css
.eds-kpi-row {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 16px;
  margin-bottom: 24px;
}

.eds-kpi-card {
  background-color: var(--eds-surface);
  border: 1px solid var(--eds-border);
  border-radius: var(--eds-radius-rounded, 4px);
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.eds-kpi-label {
  font-size: 0.75rem;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: var(--eds-text-secondary);
}

.eds-kpi-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: var(--eds-text-primary);
  font-variant-numeric: tabular-nums;
}

.eds-kpi-trend {
  font-size: 0.75rem;
  font-weight: 600;
}
.eds-kpi-trend-up { color: var(--eds-success); }
.eds-kpi-trend-down { color: var(--eds-danger); }
.eds-kpi-trend-neutral { color: var(--eds-text-secondary); }

.eds-grid-2col {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(400px, 1fr));
  gap: 16px;
}

.eds-grid-3col {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 16px;
}

@media (max-width: 768px) {
  .eds-grid-2col,
  .eds-grid-3col {
    grid-template-columns: 1fr;
  }
}
```

---

## 3. Data View Layout

### 3.1 Toolbar + Table + Pagination

```html
<div class="eds-data-view">
  <div class="eds-data-toolbar">
    <h2>Data Table</h2>
    <div class="eds-data-toolbar-actions">
      <div class="eds-search">
        <span>🔍</span>
        <input type="text" placeholder="Search..." />
      </div>
      <button class="eds-btn eds-btn-outlined">Export</button>
      <button class="eds-btn eds-btn-primary">Add New</button>
    </div>
  </div>
  <div class="eds-table-wrapper">
    <table class="eds-table">
      <!-- Table content -->
    </table>
  </div>
  <div class="eds-pagination">
    <span class="eds-pagination-info">Showing 1–10 of 247</span>
    <div class="eds-pagination-controls">
      <button class="eds-btn-ghost" disabled>Previous</button>
      <button class="eds-btn-ghost">Next</button>
    </div>
  </div>
</div>
```

```css
.eds-data-view {
  background-color: var(--eds-surface);
  border: 1px solid var(--eds-border);
  border-radius: var(--eds-radius-rounded, 4px);
  overflow: hidden;
}

.eds-data-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px;
  border-bottom: 1px solid var(--eds-border);
  flex-wrap: wrap;
  gap: 12px;
}

.eds-data-toolbar h2 {
  font-size: 1.125rem;
  font-weight: 700;
  margin: 0;
}

.eds-data-toolbar-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.eds-table-wrapper {
  overflow-x: auto;
}

.eds-pagination {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  border-top: 1px solid var(--eds-border);
  font-size: 0.875rem;
  color: var(--eds-text-secondary);
}

.eds-pagination-controls {
  display: flex;
  gap: 8px;
}
```

---

## 4. Chat Interface Layout

### 4.1 Full-Page Chat

```css
.eds-chat-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  max-width: 900px;
  margin: 0 auto;
}

.eds-chat-header {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  background-color: var(--eds-north-sea);
  color: var(--eds-white);
  flex-shrink: 0;
}

.eds-chat-header-title {
  font-size: 1.125rem;
  font-weight: 600;
}

.eds-chat-bot-icon {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background-color: var(--eds-primary);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.875rem;
  font-weight: 700;
  color: var(--eds-white);
  flex-shrink: 0;
}

.eds-chat-messages {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.eds-msg-user {
  background-color: var(--eds-primary);
  color: var(--eds-white);
  border-radius: 16px 16px 4px 16px;
  padding: 10px 16px;
  max-width: 80%;
  margin-left: auto;
  font-size: 0.9375rem;
  line-height: 1.5;
  word-wrap: break-word;
}

.eds-msg-assistant {
  background-color: var(--eds-surface-alt);
  color: var(--eds-text-primary);
  border: 1px solid var(--eds-border);
  border-radius: 16px 16px 16px 4px;
  padding: 12px 16px;
  max-width: 85%;
  font-size: 0.9375rem;
  line-height: 1.6;
  word-wrap: break-word;
}

.eds-ai-badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 8px;
  border-radius: 4px;
  background-color: var(--eds-moss-green-light);
  color: var(--eds-primary);
  font-size: 0.7rem;
  font-weight: 600;
  margin-bottom: 8px;
}

.eds-msg-assistant table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.8rem;
  margin: 8px 0;
}
.eds-msg-assistant th {
  background-color: var(--eds-primary);
  color: white;
  padding: 6px 10px;
  text-align: left;
  font-size: 0.75rem;
  font-weight: 600;
}
.eds-msg-assistant td {
  padding: 5px 10px;
  border-bottom: 1px solid var(--eds-border);
}

.eds-msg-assistant pre {
  background-color: var(--eds-north-sea);
  color: #E6E6E6;
  padding: 12px 16px;
  border-radius: 4px;
  overflow-x: auto;
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;
  font-size: 0.8rem;
  line-height: 1.5;
  margin: 8px 0;
}
.eds-msg-assistant code {
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;
  font-size: 0.85em;
  background-color: rgba(0, 112, 121, 0.08);
  padding: 2px 6px;
  border-radius: 3px;
}

.eds-chat-input-area {
  padding: 12px 16px;
  border-top: 1px solid var(--eds-border);
  display: flex;
  align-items: flex-end;
  gap: 8px;
  flex-shrink: 0;
  background-color: var(--eds-surface);
}

.eds-chat-input {
  flex: 1;
  padding: 10px 16px;
  border: 1px solid var(--eds-border);
  border-radius: 20px;
  font-family: inherit;
  font-size: 0.9375rem;
  color: var(--eds-text-primary);
  background-color: var(--eds-surface);
  resize: none;
  max-height: 120px;
  transition: border-color 0.15s ease;
}
.eds-chat-input:focus {
  outline: none;
  border-color: var(--eds-primary);
  box-shadow: 0 0 0 2px rgba(0, 112, 121, 0.2);
}
.eds-chat-input::placeholder { color: var(--eds-text-disabled); }

.eds-chat-send-btn {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  border: none;
  background-color: var(--eds-primary);
  color: var(--eds-white);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background-color 0.15s ease;
  flex-shrink: 0;
}
.eds-chat-send-btn:hover { background-color: var(--eds-primary-hover); }
.eds-chat-send-btn:disabled {
  background-color: var(--eds-medium-gray);
  cursor: not-allowed;
}

.eds-prompt-container {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  padding: 16px;
  justify-content: center;
}

.eds-prompt-chip {
  padding: 8px 16px;
  border: 1px solid var(--eds-primary);
  border-radius: 20px;
  background-color: transparent;
  color: var(--eds-primary);
  font-family: inherit;
  font-size: 0.85rem;
  cursor: pointer;
  transition: all 0.15s ease;
  text-align: left;
  line-height: 1.4;
}
.eds-prompt-chip:hover {
  background-color: var(--eds-primary);
  color: var(--eds-white);
}
```

---

## 5. Form Layout

### 5.1 Vertical Form

```css
.eds-form {
  display: flex;
  flex-direction: column;
  gap: 20px;
  max-width: 600px;
}

.eds-form-section {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.eds-form-section-title {
  font-size: 1rem;
  font-weight: 700;
  color: var(--eds-text-primary);
  padding-bottom: 8px;
  border-bottom: 2px solid var(--eds-primary);
}

.eds-form-row {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 16px;
}

.eds-form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  padding-top: 16px;
  border-top: 1px solid var(--eds-border);
}
```

---

## 6. Detail / Settings Page

```css
.eds-detail-page {
  max-width: 800px;
  margin: 0 auto;
  padding: 32px 24px;
}

.eds-detail-header {
  margin-bottom: 32px;
}

.eds-detail-header h1 {
  font-size: 1.5rem;
  font-weight: 700;
  margin-bottom: 8px;
}

.eds-detail-header p {
  font-size: 0.9375rem;
  color: var(--eds-text-secondary);
  line-height: 1.6;
}

.eds-detail-section {
  margin-bottom: 32px;
}

.eds-detail-section h2 {
  font-size: 1.125rem;
  font-weight: 700;
  margin-bottom: 16px;
  padding-bottom: 8px;
  border-bottom: 2px solid var(--eds-primary);
}

.eds-detail-field {
  display: flex;
  flex-direction: column;
  gap: 4px;
  margin-bottom: 16px;
}

.eds-detail-field-label {
  font-size: 0.75rem;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: var(--eds-text-secondary);
}

.eds-detail-field-value {
  font-size: 1rem;
  color: var(--eds-text-primary);
}
```

---

## 7. Empty State

```css
.eds-empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 64px 24px;
  text-align: center;
  gap: 16px;
}

.eds-empty-state-icon {
  width: 64px;
  height: 64px;
  border-radius: 50%;
  background-color: var(--eds-moss-green-light);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.5rem;
  color: var(--eds-primary);
}

.eds-empty-state-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: var(--eds-text-primary);
}

.eds-empty-state-description {
  font-size: 0.875rem;
  color: var(--eds-text-secondary);
  max-width: 400px;
  line-height: 1.6;
}
```

---

## 8. Responsive Utilities

```css
@media (max-width: 768px) {
  .eds-hide-mobile { display: none !important; }
}

@media (min-width: 769px) {
  .eds-hide-desktop { display: none !important; }
}

@media (max-width: 768px) {
  .eds-text-center-mobile { text-align: center; }
}

.eds-stack-mobile {
  display: flex;
  gap: 16px;
}
@media (max-width: 768px) {
  .eds-stack-mobile {
    flex-direction: column;
  }
}
```

---

## 9. Complete Starter Template

Copy this as the starting point for any new EDS-styled HTML page:

```html
<!DOCTYPE html>
<html lang="en" data-color-scheme="light">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>App Name — Equinor</title>
  <meta name="theme-color" content="#243746" />
  <link rel="stylesheet" href="https://cdn.eds.equinor.com/font/eds-uprights-vf.css" />
  <link rel="icon" href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><circle cx='50' cy='50' r='45' fill='%23007079'/><text x='50' y='65' text-anchor='middle' fill='white' font-size='40' font-weight='bold' font-family='Arial'>A</text></svg>" />
  <style>
    /* Paste eds-foundation CSS variables here */
    /* Paste eds-components CSS here */
    /* Paste eds-layouts CSS here */
  </style>
</head>
<body>
  <header class="eds-topbar">
    <img src="https://cdn.eds.equinor.com/logo/equinor-logo-horizontal.svg#white"
         alt="Equinor" class="eds-topbar-logo" />
    <span class="eds-topbar-title">App Name</span>
    <div class="eds-topbar-spacer"></div>
  </header>
  <main class="eds-page-content">
    <h1>Welcome</h1>
    <p>Your content here.</p>
  </main>
</body>
</html>
```
