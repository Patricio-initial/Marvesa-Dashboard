# CLAUDE.md — Carboclor Dashboard

This file provides guidance for AI assistants working on this codebase.

## Project Overview

**carboclor-dashboard** is a real-time tank inventory dashboard for Carboclor. It displays current stock levels, tank capacities, and occupancy percentages pulled live from a Google Sheets CSV feed.

- **Language:** Spanish (Argentine locale, `es-AR`)
- **Stack:** Pure vanilla HTML/CSS/JavaScript — no build tools, no npm, no frameworks
- **Deployment:** Single static file (`index.html`) — can be hosted on any static file server or opened directly in a browser

---

## Repository Structure

```
carboclor-dashboard/
├── index.html      # Entire application (HTML + CSS + JS in one file)
├── README.md       # Minimal readme
└── CLAUDE.md       # This file
```

There is **no build system**, no `package.json`, no `node_modules`, and no compilation step. The entire application lives in `index.html`.

---

## Application Architecture

`index.html` is organized into five clearly commented sections:

| Section | Lines | Purpose |
|---|---|---|
| `<style>` | 7–196 | All CSS (variables, layout, components) |
| `<body>` markup | 198–253 | HTML structure (header, grid, table, footer) |
| `// === CONFIG ===` | 256–258 | Hardcoded CSV URL and refresh interval |
| `// === HELPERS ===` | 260–392 | Pure utility functions |
| `// === RENDER ===` | 394–463 | DOM rendering functions |
| `// === DATA LOAD ===` | 476–519 | CSV fetch and data parsing |
| `// === EVENTS ===` | 521–533 | Event listeners and boot |

---

## Key Configuration (hardcoded in `index.html`)

```javascript
// Line 257 — Google Sheets CSV endpoint
const CSV_URL = "https://docs.google.com/spreadsheets/d/e/.../pub?...&output=csv";

// Line 258 — Auto-refresh interval
const AUTO_REFRESH_MS = 5 * 60 * 1000; // 5 minutes
```

To change the data source or refresh interval, edit these two constants directly in `index.html`. There is no `.env` file or external config.

---

## Data Flow

```
Google Sheets (published CSV)
        ↓
  fetch() with anti-cache timestamp param (?_ts=...)
        ↓
  parseCSV(text)       — custom CSV parser (handles quoted commas/newlines)
        ↓
  normalizeHeaders()   — maps flexible column names to canonical keys
        ↓
  parseKg() / parsePercent()  — handle mixed number formats (1.234,56 / 1,234.56)
        ↓
  filterData() → sortData()
        ↓
  render()             — writes cards + table rows via innerHTML / createElement
```

### Expected CSV Columns

The sheet must have these columns (case-insensitive, various naming conventions accepted):

| Canonical key | Accepted header variants |
|---|---|
| `tank` | `tank`, `tanque` |
| `supplier` | `product/supplier`, `proveedor`, `producto/proveedor`, `product`, `supplier` |
| `stockKg` | `stock_ kg`, `stocl_ kg`, `stock kg`, `stock_kg`, `stock`, `kg`, `kgs` |
| `capKg` | `capacity_kg`, `capacity kg`, `capacidad_kg`, `capacidad kg`, `capacity` |
| `pct` | `% used`, `%used`, `used`, `ocupacion`, `% ocupación`, `% ocupacion` |

If column mapping fails, the app shows a Spanish error message and lists the expected header names.

---

## CSS Design System

### CSS Custom Properties (`:root`)

```css
--bg:     #0b1020   /* Page background */
--card:   #111a33   /* Card background */
--muted:  #9fb0d0   /* Secondary text */
--text:   #e9f0ff   /* Primary text */
--line:   rgba(255,255,255,.08)  /* Borders/dividers */
--shadow: 0 12px 30px rgba(0,0,0,.35)
--radius: 16px
```

Always use these variables for colors and borders. Do not hardcode color values for structural elements.

### Layout

- **Grid:** 12-column CSS Grid (`grid-template-columns: repeat(12, 1fr)`)
- **Cards:** `grid-column: span 6` on desktop, `span 12` on mobile (`max-width: 920px`)
- **Responsive breakpoint:** 920px (single media query in `index.html:85`)

### Status Color System

Occupancy-based status with consistent color tokens:

| Status | Threshold | Color |
|---|---|---|
| OK | 0–69% | `rgba(52,211,153,.95)` — green |
| Alto | 70–89% | `rgba(245,158,11,.95)` — amber |
| Crítico | 90–99% | `rgba(239,68,68,.95)` — red |
| Full | ≥100% | `rgba(168,85,247,.95)` — purple |
| Sin dato | null | `rgba(148,163,184,.9)` — gray |

Always use `statusForPct(pct)` (`index.html:362`) to derive both label and color — never hardcode status colors inline.

---

## Key Functions Reference

| Function | Location | Description |
|---|---|---|
| `parseCSV(text)` | line 294 | Full CSV parser: handles quoted cells, escaped `""`, CRLF |
| `parseKg(value)` | line 275 | Parses `1,234.56` and `1.234,56` formats to float |
| `parsePercent(value)` | line 268 | Strips `%`, normalizes comma→dot, returns float or `null` |
| `normalizeHeaders(headers)` | line 340 | Maps raw header strings to `{tank, supplier, stockKg, capKg, pct}` index map |
| `statusForPct(pct)` | line 362 | Returns `{label, color}` for a given occupancy percentage |
| `sortData(data, mode)` | line 370 | Sorts array by one of 6 modes (pct_desc/asc, tank_asc/desc, kg_desc/asc) |
| `filterData(data, q)` | line 385 | Case-insensitive search across `tank` + `supplier` fields |
| `render(data)` | line 395 | Rebuilds entire card grid and table from data array |
| `clampPct(pct)` | line 465 | Clamps value to 0–100 for CSS `width` |
| `escapeHtml(s)` | line 470 | XSS-safe HTML escaping for user-facing strings |
| `load()` | line 477 | Full fetch→parse→render cycle; also updates timestamp and error state |
| `formatNumber(n)` | line 263 | Formats numbers with `es-AR` locale (comma thousands, dot decimals) |

---

## Development Conventions

### Code Style

- **No semicolons** are consistently used after block statements; semicolons used for single-line statements
- **Arrow functions** preferred for short callbacks and comparators
- **`const`** for configuration and DOM references; `let` only where reassignment is needed
- **Section comments** (`// === SECTION ===`) separate logical blocks — maintain this structure when adding code
- Helper `const $ = (id) => document.getElementById(id)` is the only DOM shorthand; use it for all `getElementById` calls
- All user-visible strings are in **Spanish**; keep new UI strings in Spanish

### Security

- All dynamic content inserted into the DOM must go through `escapeHtml()` to prevent XSS
- The CSV data is untrusted external input — always escape before rendering
- Never use `eval()` or `new Function()` with data from the CSV

### Number Formatting

- **Display:** always use `formatNumber(n)` which applies `Intl.NumberFormat("es-AR")`
- **Parsing:** use `parseKg()` for weight values, `parsePercent()` for percentages
- Never call `parseFloat()` directly on raw CSV cell values — the format may be ambiguous

### Error Handling

- Errors during `load()` are caught and displayed in `#errorBox` in Spanish
- The loading indicator (`#loading`) must be hidden in the `finally` block — do not add early returns that bypass it
- Column mapping errors should list the expected header names to aid debugging

### Performance

- Search input uses 180ms debounce — do not reduce this
- Anti-cache busting via `?_ts=${Date.now()}` and `cache: "no-store"` is intentional; do not remove it
- DOM updates in `render()` use `createElement` + `appendChild` for cards and `innerHTML` is cleared once per render cycle — keep bulk updates together

---

## No Build / No Test Infrastructure

This project has **no test suite, no linter, and no CI**. There is no `npm test`, `npm run build`, or similar command. Verification is manual:

1. Open `index.html` directly in a browser (or serve it with any static server, e.g. `python3 -m http.server`)
2. Confirm the dashboard loads and shows data from Google Sheets
3. Test the search filter, sort dropdown, and Refrescar button

---

## Git Workflow

- **Main branch:** `main` (remote `origin`)
- **Feature branches:** use the `claude/` prefix for AI-assisted work
- Commits historically follow a simple pattern: descriptive message for the changed file (e.g., `Update index.html`)
- There is no PR template or required review process

---

## Common Tasks

### Change the Google Sheets data source

Edit `CSV_URL` on `index.html:257`. The new sheet must be published as CSV via **File → Share → Publish to web → CSV**.

### Add a new sort option

1. Add an `<option value="new_mode">` to `#sort` in the HTML markup
2. Add the comparator to the `cmp` object in `sortData()` at `index.html:372`

### Add a new status level

Edit `statusForPct()` at `index.html:362`. Add a new threshold check and return a new `{label, color}` object.

### Change the auto-refresh interval

Edit `AUTO_REFRESH_MS` on `index.html:258`.

### Add a new displayed field

1. Extend the data mapping in `load()` at `index.html:497` to extract the new field from the CSV row
2. Update `normalizeHeaders()` at `index.html:340` to recognize the new column's header variants
3. Add the field to the `render()` function at `index.html:395` (both card and table row templates)
