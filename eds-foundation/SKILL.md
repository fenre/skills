---
name: eds-foundation
description: Applies Equinor Design System (EDS) foundation tokens — colors, typography, spacing, elevation, shape, motion, and accessibility — to any web output including Databricks Apps, notebook HTML, dashboards, and standalone web interfaces. Uses a CSS-only approach with the EDS CDN font (no React or npm dependency). ALWAYS use this skill whenever the user asks to use "Equinor colors", "Equinor color scheme", "EDS colors", "EDS color scheme", "the Equinor palette", "Equinor's colors", or any variation referring to Equinor color usage, color palette, or color scheme — this is a mandatory trigger, even for small CSS snippets or single-color questions. Also use when the user mentions "Equinor styling", "EDS", "Equinor brand", "corporate styling", "Atlas look and feel", "Norwegian Woods", "North Sea", "Energy Red", or any EDS color name; when creating any HTML, CSS, or web-based output that should follow Equinor visual standards; when making a UI look like an Equinor internal application; or when asked about colors, fonts, spacing, or visual identity for an Equinor application. Always apply this skill BEFORE eds-components or eds-layouts skills.
---

# Equinor Design System — Foundation

## Description

Apply Equinor Design System (EDS) foundation tokens to any web output: Databricks Apps,
notebook HTML, dashboards, or standalone web interfaces. This skill provides the core
visual identity layer — colors, typography, spacing, elevation, and shape — that every
EDS-compliant interface requires.

## When to apply

- **MANDATORY trigger**: Any time the user asks to use "Equinor colors", "Equinor color
  scheme", "EDS colors", "EDS color scheme", "the Equinor palette", or any variation
  referring to Equinor color usage — apply this skill immediately, even for small
  snippets or single-color answers. Never just recite a hex value; always ground the
  answer in the EDS palette, semantic aliases, and CSS variables defined below.
- User mentions "Equinor styling", "EDS", "brand", "Equinor design", "corporate styling",
  or "Atlas look and feel"
- User asks about colors, fonts, spacing, or visual identity for an Equinor application
- User wants to make any UI look like an Equinor internal application
- User references "Norwegian Woods", "North Sea", "Energy Red", or any EDS color name
- User is creating any HTML, CSS, or web-based output that should follow Equinor standards
- Always apply this skill BEFORE the eds-components or eds-layouts skills

## Important context

This skill uses a **CSS-only approach** — replicating the EDS visual identity using plain
CSS custom properties and the EDS CDN font. This avoids the `@equinor/eds-core-react`
dependency (which requires React 19 and `styled-components`) and works in any HTML context
including Databricks notebooks, Databricks Apps, and standalone web pages.

For projects that DO use React and npm, the official packages are:
- `@equinor/eds-core-react` — React component library
- `@equinor/eds-tokens` — Design tokens (JS objects + CSS variables)
- `@equinor/eds-icons` — Icon library
- `@equinor/eds-tailwind` — Tailwind CSS plugin
- `@equinor/eds-data-grid-react` — Data grid (AG Grid wrapper)

Official references:
- Documentation: https://eds.equinor.com
- GitHub: https://github.com/equinor/design-system
- Storybook: https://storybook.eds.equinor.com
- npm: https://www.npmjs.com/package/@equinor/eds-tokens

---

## 1. Font Loading

The Equinor typeface is proprietary and loaded from the EDS CDN. Add ONE of these to
the `<head>` of your HTML document. The variable font is the recommended option:

```html
<!-- RECOMMENDED: Variable font (supports all weights, required for EDS 2.0) -->
<link rel="stylesheet" href="https://cdn.eds.equinor.com/font/eds-uprights-vf.css" />

<!-- Alternative: Regular weight only (smaller download) -->
<link rel="stylesheet" href="https://cdn.eds.equinor.com/font/equinor-regular.css" />

<!-- Legacy: Older font file (includes only Equinor typeface) -->
<link rel="stylesheet" href="https://cdn.eds.equinor.com/font/equinor-font.css" />
```

Set the body font stack with system font fallbacks:

```css
body {
  font-family: 'Equinor', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto,
    Oxygen, Ubuntu, Cantarell, 'Helvetica Neue', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
```

### Font weights

The Equinor typeface has four weights. Do NOT use `light` in digital interfaces unless
the font size is over 48px:

| Weight name | CSS `font-weight` | Usage |
|---|---|---|
| Light | 300 | Special cases only (>48px font size) |
| Regular | 400 | Body text, descriptions, captions |
| Medium | 500 | Subheadings, emphasis, navigation |
| Bold | 700 | Headings, strong emphasis, CTAs |

---

## 2. Color System

### 2.1 Brand Colors

These are Equinor's core brand colors. They define the identity and should be used
consistently across all interfaces.

| Name | Hex | RGB | CSS Variable | Usage |
|---|---|---|---|---|
| Energy Red | `#FF1243` | 255, 18, 67 | `--eds-energy-red` | Logo, critical alerts, primary CTA accents |
| North Sea | `#243746` | 36, 55, 70 | `--eds-north-sea` | Headers, dark backgrounds, navigation |
| Norwegian Woods | `#007079` | 0, 112, 121 | `--eds-norwegian-woods` | Primary interactive: buttons, links, focus |

### 2.2 Extended Palette

| Name | Hex | CSS Variable | Usage |
|---|---|---|---|
| Teal Light | `#008489` | `--eds-teal-light` | Hover state for Norwegian Woods |
| Teal Dark | `#004F55` | `--eds-teal-dark` | Active/pressed state |
| Moss Green Light | `#DEEDEE` | `--eds-moss-green-light` | Subtle highlights, info badges |
| Moss Green Medium | `#C3DCDC` | `--eds-moss-green-medium` | Borders on highlighted elements |
| Mist Blue | `#D5EAF4` | `--eds-mist-blue` | Secondary backgrounds, info panels |
| Spruce Wood | `#FF7D50` | `--eds-spruce-wood` | Warning accents (use sparingly) |
| Heritage Red | `#EB0037` | `--eds-heritage-red` | Errors, destructive actions |

### 2.3 Neutral Colors

| Name | Hex | CSS Variable | Usage |
|---|---|---|---|
| White | `#FFFFFF` | `--eds-white` | Page backgrounds, card backgrounds |
| Off-white | `#F7F7F7` | `--eds-off-white` | Alternate backgrounds, subtle surfaces |
| Light Gray | `#E6E6E6` | `--eds-light-gray` | Borders, dividers, input borders |
| Medium Gray | `#BEBEBE` | `--eds-medium-gray` | Disabled text, placeholder text |
| Dark Gray | `#6F6F6F` | `--eds-dark-gray` | Secondary text, captions, metadata |
| Text Primary | `#3D3D3D` | `--eds-text` | Body text, headings (NOT pure black) |
| Black | `#000000` | — | Avoid for text; use `#3D3D3D` instead |

### 2.4 Semantic Color Aliases

These semantic aliases map to the palette above and should be used in component
styling rather than raw color values:

| Alias | Maps to | Purpose |
|---|---|---|
| `--eds-primary` | Norwegian Woods `#007079` | Primary interactive color |
| `--eds-primary-hover` | Teal Light `#008489` | Hover state |
| `--eds-primary-active` | Teal Dark `#004F55` | Active/pressed state |
| `--eds-danger` | Energy Red `#FF1243` | Destructive actions, critical alerts |
| `--eds-danger-dark` | Heritage Red `#EB0037` | Error states |
| `--eds-warning` | Spruce Wood `#FF7D50` | Warning states |
| `--eds-success` | `#4BB748` | Success states |
| `--eds-info` | Mist Blue `#D5EAF4` | Informational states |
| `--eds-surface` | White `#FFFFFF` | Primary surface |
| `--eds-surface-alt` | Off-white `#F7F7F7` | Alternate surface |
| `--eds-border` | Light Gray `#E6E6E6` | Default border |
| `--eds-text-primary` | `#3D3D3D` | Primary text |
| `--eds-text-secondary` | Dark Gray `#6F6F6F` | Secondary text |
| `--eds-text-disabled` | Medium Gray `#BEBEBE` | Disabled text |

### 2.5 New EDS Token System (CSS Custom Properties)

The EDS tokens package (v2+) introduces a semantic CSS variable system that
automatically adapts to light and dark color schemes using the `light-dark()` function.
These variables are synced from Figma and represent the current official token naming.

If you are using `@equinor/eds-tokens` via npm, import variables with:

```css
@import '@equinor/eds-tokens/css/variables';
```

Key semantic variable examples (static approach):

```css
/* Backgrounds */
var(--eds-color-bg-neutral-surface)       /* Light: #ffffff, Dark: #262626 */
var(--eds-color-bg-accent-emphasis)       /* Accent emphasis background */

/* Text */
var(--eds-color-text-neutral-strong)      /* Primary text */
var(--eds-color-text-accent-strong-on-emphasis) /* Text on accent emphasis bg */

/* Borders */
var(--eds-color-border-neutral-subtle)    /* Subtle neutral border */
var(--eds-color-border-info-medium)       /* Info-colored border */
```

For the dynamic approach, use `data-color-appearance` attributes:

```html
<button data-color-appearance="accent">Continue</button>
<div data-color-appearance="danger">Error message</div>
```

With abstract variables:

```css
.button {
  background: var(--eds-color-bg-emphasis);
  color: var(--eds-color-text-strong-on-emphasis);
}
```

Appearance values: `neutral` (default), `accent`, `success`, `info`, `warning`, `danger`.

### 2.6 Complete CSS Variables Block

For projects NOT using npm, paste this complete block into your root stylesheet.
This replicates the EDS visual identity using plain CSS:

```css
:root {
  /* ── Brand ── */
  --eds-energy-red: #FF1243;
  --eds-north-sea: #243746;
  --eds-norwegian-woods: #007079;

  /* ── Interactive states ── */
  --eds-teal-light: #008489;
  --eds-teal-dark: #004F55;

  /* ── Extended palette ── */
  --eds-moss-green-light: #DEEDEE;
  --eds-moss-green-medium: #C3DCDC;
  --eds-mist-blue: #D5EAF4;
  --eds-spruce-wood: #FF7D50;
  --eds-heritage-red: #EB0037;

  /* ── Neutrals ── */
  --eds-white: #FFFFFF;
  --eds-off-white: #F7F7F7;
  --eds-light-gray: #E6E6E6;
  --eds-medium-gray: #BEBEBE;
  --eds-dark-gray: #6F6F6F;
  --eds-text: #3D3D3D;

  /* ── Semantic aliases ── */
  --eds-primary: var(--eds-norwegian-woods);
  --eds-primary-hover: var(--eds-teal-light);
  --eds-primary-active: var(--eds-teal-dark);
  --eds-danger: var(--eds-energy-red);
  --eds-danger-dark: var(--eds-heritage-red);
  --eds-warning: var(--eds-spruce-wood);
  --eds-success: #4BB748;
  --eds-info: var(--eds-mist-blue);
  --eds-surface: var(--eds-white);
  --eds-surface-alt: var(--eds-off-white);
  --eds-border: var(--eds-light-gray);
  --eds-text-primary: var(--eds-text);
  --eds-text-secondary: var(--eds-dark-gray);
  --eds-text-disabled: var(--eds-medium-gray);
}
```

### 2.7 Data Visualization Colors

When creating charts and graphs, use these colors in order for categorical data:

1. Norwegian Woods `#007079` (primary series)
2. Energy Red `#FF1243`
3. Mist Blue `#5098B3`
4. Spruce Wood `#FF7D50`
5. Moss Green Medium `#7FBFBF`
6. North Sea `#243746`

For sequential/gradient data, use a single-hue ramp from Moss Green Light to
Norwegian Woods to North Sea.

---

## 3. Typography Scale

### 3.1 Headings

| Style | Font Size | Font Weight | Line Height | Letter Spacing |
|---|---|---|---|---|
| H1 Bold | 2rem (32px) | 700 | 1.25 | 0 |
| H1 | 2rem (32px) | 400 | 1.25 | 0 |
| H2 | 1.5rem (24px) | 700 | 1.33 | 0 |
| H3 | 1.25rem (20px) | 700 | 1.4 | 0 |
| H4 | 1.125rem (18px) | 700 | 1.33 | 0 |
| H5 | 1rem (16px) | 700 | 1.5 | 0 |
| H6 | 0.875rem (14px) | 700 | 1.43 | 0 |

### 3.2 Paragraph Styles

| Style | Font Size | Font Weight | Line Height | Usage |
|---|---|---|---|---|
| Overline | 0.625rem (10px) | 500 | 1.6 | Labels above sections |
| Ingress | 1.125rem (18px) | 400 | 1.55 | Introductory paragraphs |
| Body Long | 1rem (16px) | 400 | 1.6 | Long-form content |
| Body Short | 0.875rem (14px) | 400 | 1.43 | Short labels in components |
| Caption | 0.75rem (12px) | 500 | 1.33 | Captions, metadata, timestamps |

### 3.3 Typography Rules

- Use **sentence case** (The quick brown fox) not title case (The Quick Brown Fox)
- Line length: 55–80 characters for readability
- Never use the Light weight (300) below 48px
- Use Body Short for text within components (four words or less)
- Use Body Long for paragraphs and descriptions

### 3.4 CSS Implementation

```css
h1       { font-size: 2rem;    font-weight: 700; line-height: 1.25; }
h2       { font-size: 1.5rem;  font-weight: 700; line-height: 1.33; }
h3       { font-size: 1.25rem; font-weight: 700; line-height: 1.4;  }
h4       { font-size: 1.125rem; font-weight: 700; line-height: 1.33; }
h5       { font-size: 1rem;    font-weight: 700; line-height: 1.5;  }
h6       { font-size: 0.875rem; font-weight: 700; line-height: 1.43; }

body, p  { font-size: 1rem;    font-weight: 400; line-height: 1.6; color: var(--eds-text); }

.eds-overline   { font-size: 0.625rem; font-weight: 500; line-height: 1.6;
                  text-transform: uppercase; letter-spacing: 0.05em; }
.eds-ingress    { font-size: 1.125rem; font-weight: 400; line-height: 1.55; }
.eds-body-short { font-size: 0.875rem; font-weight: 400; line-height: 1.43; }
.eds-caption    { font-size: 0.75rem;  font-weight: 500; line-height: 1.33;
                  color: var(--eds-text-secondary); }
```

---

## 4. Spacing System

EDS uses a 4px base grid with an 8px standard increment:

| Token | Value | CSS Variable | Usage |
|---|---|---|---|
| 1 | 4px | `--eds-spacing-1` | Tight gaps, icon padding |
| 2 | 8px | `--eds-spacing-2` | Default gap, small padding |
| 3 | 12px | `--eds-spacing-3` | Medium padding |
| 4 | 16px | `--eds-spacing-4` | Standard padding, section gaps |
| 5 | 24px | `--eds-spacing-5` | Large section gaps |
| 6 | 32px | `--eds-spacing-6` | Page section spacing |
| 8 | 48px | `--eds-spacing-8` | Major section breaks |

```css
:root {
  --eds-spacing-1: 4px;
  --eds-spacing-2: 8px;
  --eds-spacing-3: 12px;
  --eds-spacing-4: 16px;
  --eds-spacing-5: 24px;
  --eds-spacing-6: 32px;
  --eds-spacing-8: 48px;
}
```

### EDS 2.0 Spacing (data attributes)

The newer token system uses data attributes for contextual spacing:

```css
/* Selectable elements (buttons, list items) */
.button {
  padding-inline: var(--eds-selectable-space-horizontal);
  padding-block: var(--eds-selectable-space-vertical);
  gap: var(--eds-selectable-gap-vertical) var(--eds-selectable-gap-horizontal);
}

/* Container elements (cards, panels) */
.card {
  padding-inline: var(--eds-container-space-horizontal);
  padding-block: var(--eds-container-space-vertical);
  gap: var(--eds-container-gap-vertical) var(--eds-container-gap-horizontal);
}

/* Page-level spacing */
.page {
  padding-inline: var(--eds-page-space-horizontal);
  padding-block: var(--eds-page-space-vertical);
  gap: var(--eds-page-gap-vertical) var(--eds-page-gap-horizontal);
}
```

Space sizes via `data-selectable-space` or `data-container-space`: `xs`, `sm`, `md`, `lg`, `xl`.
Space proportions via `data-space-proportions`: `squished`, `squared`, `stretched`.

---

## 5. Elevation (Shadows)

| Level | Box Shadow | Usage |
|---|---|---|
| None | `none` | Flat elements |
| Raised | `0 1px 5px rgba(0,0,0,0.12), 0 2px 2px rgba(0,0,0,0.08)` | Cards, panels |
| Overlay | `0 4px 8px rgba(0,0,0,0.16), 0 2px 4px rgba(0,0,0,0.08)` | Dropdowns, popovers |
| Sticky | `0 6px 16px rgba(0,0,0,0.20), 0 4px 8px rgba(0,0,0,0.08)` | Sticky headers, modals |
| Temporary | `0 12px 24px rgba(0,0,0,0.24), 0 8px 16px rgba(0,0,0,0.08)` | Dialogs, side sheets |

```css
:root {
  --eds-elevation-none: none;
  --eds-elevation-raised: 0 1px 5px rgba(0,0,0,0.12), 0 2px 2px rgba(0,0,0,0.08);
  --eds-elevation-overlay: 0 4px 8px rgba(0,0,0,0.16), 0 2px 4px rgba(0,0,0,0.08);
  --eds-elevation-sticky: 0 6px 16px rgba(0,0,0,0.20), 0 4px 8px rgba(0,0,0,0.08);
  --eds-elevation-temporary: 0 12px 24px rgba(0,0,0,0.24), 0 8px 16px rgba(0,0,0,0.08);
}
```

---

## 6. Shape (Border Radius)

| Token | Value | Usage |
|---|---|---|
| None | `0px` | Tables, full-width elements |
| Subtle | `2px` | Inputs, text fields |
| Rounded | `4px` | Cards, buttons, badges |
| Pill | `9999px` | Chips, pills, toggle switches |
| Circle | `50%` | Avatars, icon containers |

```css
:root {
  --eds-radius-none: 0px;
  --eds-radius-subtle: 2px;
  --eds-radius-rounded: 4px;
  --eds-radius-pill: 9999px;
  --eds-radius-circle: 50%;
}
```

---

## 7. Logo Assets

The Equinor logo is available from the EDS CDN in two layouts and three colors:

```html
<!-- Primary logo (stacked) -->
<img src="https://cdn.eds.equinor.com/logo/equinor-logo-primary.svg#red" alt="Equinor" />
<img src="https://cdn.eds.equinor.com/logo/equinor-logo-primary.svg#white" alt="Equinor" />
<img src="https://cdn.eds.equinor.com/logo/equinor-logo-primary.svg#black" alt="Equinor" />

<!-- Horizontal logo -->
<img src="https://cdn.eds.equinor.com/logo/equinor-logo-horizontal.svg#red" alt="Equinor" />
<img src="https://cdn.eds.equinor.com/logo/equinor-logo-horizontal.svg#white" alt="Equinor" />
<img src="https://cdn.eds.equinor.com/logo/equinor-logo-horizontal.svg#black" alt="Equinor" />
```

Color is selected via the URL fragment identifier (`#red`, `#white`, `#black`).
Default (no fragment) is black.

### Favicon

For quick favicon generation without a separate file, use an inline SVG data URI.
Replace "X" with your app's initial:

```html
<link rel="icon" href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><circle cx='50' cy='50' r='45' fill='%23007079'/><text x='50' y='65' text-anchor='middle' fill='white' font-size='40' font-weight='bold' font-family='Arial'>X</text></svg>" />
```

---

## 8. Page Metadata

Standard metadata for any EDS-styled application:

```html
<meta name="theme-color" content="#243746" />
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
```

---

## 9. Dark Mode

### Using data attributes (EDS 2.0)

```html
<html data-color-scheme="light">  <!-- or "dark" -->
```

### Using CSS classes (custom)

```css
.light { color-scheme: light; }
.dark  { color-scheme: dark; }
```

### Manual CSS variable override (non-npm projects)

```css
[data-color-scheme="dark"], .dark-mode {
  --eds-surface: #1A1A1A;
  --eds-surface-alt: #262626;
  --eds-border: #404040;
  --eds-text-primary: #E6E6E6;
  --eds-text-secondary: #A0A0A0;
  --eds-text-disabled: #666666;
  --eds-light-gray: #404040;
  --eds-off-white: #2A2A2A;
}
```

---

## 10. Motion / Transitions

EDS transitions are subtle and purposeful — they serve function, not decoration:

```css
:root {
  --eds-transition-fast: 0.1s ease;
  --eds-transition-normal: 0.15s ease;
  --eds-transition-slow: 0.3s ease;
}
```

Use `--eds-transition-fast` for hover states, `--eds-transition-normal` for focus and
toggles, `--eds-transition-slow` for panels and modals.

---

## 11. Accessibility Requirements

EDS targets **WCAG 2.1 AA** compliance:

- Color contrast: text on backgrounds must meet 4.5:1 ratio minimum
- Large text (18px+ bold or 24px+ regular): 3:1 ratio minimum
- Never use color alone to convey information
- All interactive elements must have visible focus indicators
- Focus indicator: `box-shadow: 0 0 0 2px rgba(0, 112, 121, 0.2)` on focus
- Use semantic HTML (`<button>`, `<input>`, `<table>`, `<nav>`)
- All buttons and inputs must be keyboard accessible

---

## 12. Global Reset

Apply this minimal reset for EDS-styled pages:

```css
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: 'Equinor', -apple-system, BlinkMacSystemFont, 'Segoe UI',
    Roboto, sans-serif;
  font-size: 1rem;
  line-height: 1.6;
  color: var(--eds-text-primary);
  background-color: var(--eds-surface);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

a { color: var(--eds-primary); text-decoration: none; }
a:hover { color: var(--eds-primary-hover); text-decoration: underline; }

::-webkit-scrollbar { width: 6px; height: 6px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb { background: var(--eds-primary); border-radius: 3px; }
::-webkit-scrollbar-thumb:hover { background: var(--eds-primary-hover); }
```
