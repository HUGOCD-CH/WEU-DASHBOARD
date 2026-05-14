# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A single-file supply chain operations dashboard for Medtronic's Western Europe (WEU) business. The entire application is one HTML file (`WEU_Operations_Dashboard.html`, ~3.9MB) containing inline CSS and ~5,700 lines of vanilla JavaScript. There is no build system, no package manager, and no server — users open the file directly in a browser.

## Development

**No build step.** Open `WEU_Operations_Dashboard.html` directly in a browser. All changes take effect on next page reload.

External dependencies are loaded from CDN at runtime:
- **Chart.js 4.4.1** — charting
- **XLSX 0.18.5** — Excel file parsing (read)
- **ExcelJS 4.3.0** — Excel file writing (export)

## Architecture

### File Layout (single HTML file)

| Lines | Content |
|-------|---------|
| ~1–804 | HTML structure — header, 5-tab navigation, pre-built DOM panels |
| ~10–461 | Inline CSS — responsive grid, dark/light theme via `body.dark-mode` |
| ~805–6492 | Vanilla JS — all logic, no modules |

### Five Main Tabs

1. **BO Dev** — Backorder development tracking (primary view)
2. **ROR** — Rate of Receipt: CFN-level demand vs. supply analysis
3. **Inventory (INV)** — SAP warehouse stock across DCs (Heerlen, Rolo, Alcala, Kits)
4. **In-Transit (STO)** — Stock Transfer Orders across regions
5. **FCST** — Monthly/quarterly demand forecasting

### Global State

All state lives in in-memory globals:

```js
ROR_DATA, INV_DATA, STO_DATA, FCST_DATA   // Parsed data arrays
DEMAND_DATA                                 // Calculated weekly demand by CFN
SCATTER_IMP, SCATTER_STA, SCATTER_WORS     // Scatter plot data splits
activeFilter                                // Current BO filter (imp/neut/wors)
_rorCountryKey                              // Selected country in ROR tab
_countryTableView                           // global vs. region-specific
_countryTableQuarter                        // Monthly vs. Quarterly toggle
cfnSelectedSet                              // Selected CFNs in global search
```

Persistence uses **IndexedDB** (uploaded file bytes) and **localStorage** (dark mode, settings).

### Key Function Families

| Family | Examples |
|--------|---------|
| Parsing | `parseRORSheet()`, `parseSTOSheet()`, `parseINVSheet()`, `_readFcstDetails()` |
| Processing | `processBO()`, `processCEMABO()`, `processInvFile()`, `processSTOFile()` |
| Rendering | `renderROR()`, `renderSTO()`, `renderInv()`, `renderFCSTTable()`, `renderBODevTable()` |
| Charting | `buildCharts()`, `buildScatterChart()`, `buildCountryCards()`, `buildFcstDemandCharts()` |
| Filtering | `filterROR()`, `filterSTO()`, `filterFCST()`, `applyScatterFilter()` |
| File I/O | `loadFromFolder()`, `idbOpen/Put/Get/LoadBytes/SaveBytes()` |
| Navigation | `showTab()`, `toggleDarkMode()` |

### Data Constants (top of JS section)

```js
WEEK_KEYS / WEEK_LABELS    // Active weeks and their display labels
_COUNTRY_FLAGS             // Emoji flags per country code
CFN_MPG_MAP                // CFN → Material Product Group
COUNTRY_GROUPS             // Country groupings for navigation
FOLDER_MATCHERS            // Patterns for auto-detecting uploaded files
```

### Special Cases to Be Aware Of

- **Israel** has dedicated data sources and separate handling throughout — it's not part of the standard WEU DC flow.
- **CEMA BO** is a separate backorder file processed by `processCEMABO()` alongside the main BO file.
- `PAGE_SIZE = 100` controls pagination across all tables.
- Weekly demand is recalculated lazily via `recalcWeeklyDemand()` and `recalcRORDevImpact()` after file loads.
- Banner state (`setBOBannerLoading()`, `setBOBannerSuccess()`) drives UX feedback during async file operations.

### File Upload Flow

Users load data via folder selection (`showOpenFilePicker()`). `FOLDER_MATCHERS` maps filename patterns to handlers (`handleBOFile`, `handleInvFallback`, `handleFCSTFallback`, etc.). Parsed bytes are cached in IndexedDB so files persist across page reloads.
