# CLAUDE.md – inflation_tool

## Project Overview

**Der Vermögensvernichter** ("The Wealth Destroyer") is a single-page, client-side inflation calculator targeting German-speaking users. It visualises how inflation erodes the real purchasing power of savings over time, comparing two account types (Girokonto vs. Tagesgeld) using live data from Eurostat and the ECB.

---

## Repository Structure

```
inflation_tool/
├── index.html   # The entire application (HTML + CSS + JS, ~1 100 lines)
└── README.md    # One-line description
```

There is **no build system, no package manager, no backend, and no test suite**. The entire app ships as a single self-contained HTML file that runs directly in any modern browser.

---

## Technology Stack

| Concern | Technology |
|---------|-----------|
| Language | Vanilla JavaScript (ES2020+), HTML5, CSS3 |
| Charts | [Chart.js 4.4.7](https://www.chartjs.org/) via CDN (`jsdelivr.net`) |
| Font | DM Sans via Google Fonts CDN |
| Inflation data | Eurostat HICP API (SDMX/JSON) |
| Interest rate data | ECB MFI Statistics API (JSON) |
| Persistence | None – all state is in-memory and resets on reload |

No `npm install`, no compilation step, no server required. Open `index.html` in a browser to run it.

---

## Application Architecture

The file is divided into three logical sections:

### 1. CSS (lines 10–555)
Embedded in `<style>`. Mobile-first responsive design with a single breakpoint at `800px`. Uses CSS Grid (two-column layout) and Flexbox extensively.

Key classes:
- `.main-layout` – two-column grid (controls 380 px | chart fills remaining space)
- `.controls` – left panel containing all sliders and toggles
- `.chart-area` – right panel with Chart.js canvas
- `.bottom-stats` – 3-column stat cards below the chart
- `.toggle-card` / `.collapsible-panel` – animated historical-data toggle section
- `.dual-range` – custom dual-handle range slider built from two overlapping `<input type="range">` elements

### 2. HTML (lines 559–697)
Semantic structure: `<header>`, controls panel, chart area, stat cards, and data-source attribution footer. All user-facing copy is in German.

### 3. JavaScript (lines 699–1094)
Plain vanilla JS, no frameworks. Organised into clearly commented sections (use the `// ──` banners as landmarks).

---

## JavaScript: Key Functions

| Function | Lines | Purpose |
|----------|-------|---------|
| `fetchOvernightRate()` | 716–746 | Fetches current German overnight deposit rate from ECB MFI API; falls back to `0.44%` on error |
| `fetchEurostat()` | 748–785 | Fetches annual HICP inflation rates (2000–2025) from Eurostat; falls back to `useFallback()` on error |
| `useFallback()` | 787–798 | Populates `eurostatData` with hardcoded 2000–2024 EA HICP values if the API is unavailable |
| `getValues()` | 820–837 | Reads all current UI state into a plain object |
| `getHistoricRates()` | 839–845 | Slices `eurostatData` for the user-selected year range |
| `getInflationRate()` | 847–852 | Returns the per-year inflation rate (historic cycled or constant) for a given year index |
| `calculate()` | 855–883 | Core calculation: builds year-by-year arrays for Girokonto and Tagesgeld real values |
| `update()` | 910–984 | Orchestrator: calls `calculate()`, updates all DOM elements, manages UI state |
| `updateChart()` | 987–1075 | Destroys and recreates the Chart.js line chart with current data |
| `updateSliderFill()` | 886–890 | Applies blue fill behind a slider thumb via inline `background` gradient |
| `updateDualRange()` | 892–907 | Syncs the dual-handle year-range slider track fill and label |
| `selectAccount()` | 811–817 | Switches the active account type and re-renders |
| `fmtEuro()` / `fmtK()` | 801–808 | Locale-aware euro formatting helpers (German locale `de-DE`) |

---

## Global State Variables

```js
let eurostatData = {};         // { year: inflationRate } populated from Eurostat/fallback
let eurostatLoaded = false;    // guard: Eurostat data is ready
let overnightRate = null;      // ECB overnight rate as a plain number, e.g. 0.44
let overnightRatePeriod = '';  // human-readable period label from ECB response
let selectedAccount = 'girokonto'; // 'girokonto' | 'tagesgeld'
let chart;                     // Chart.js instance (replaced on every update)
```

---

## API Integrations

### Eurostat HICP
- **URL constant:** `EUROSTAT_URL` (line 709)
- **Dataset:** `prc_hicp_aind` – Annual average HICP rate of change, all items, Eurozone
- **Period:** 2000–2025
- **Format:** SDMX JSON (`format=JSON`)
- **Triggered:** Lazily, only when the user first enables the "Historische Daten" toggle

### ECB MFI Interest Rate
- **URL constant:** `ECB_MIR_URL` (line 712)
- **Dataset:** `MIR` – Monthly Interest Rates, Germany, overnight deposits, private households, new business
- **Returns:** Last 1 observation (`lastNObservations=1`)
- **Triggered:** On page load (`fetchOvernightRate()` called in init block at line 1092)

Both functions use `try/catch` with silent console errors and graceful UI fallbacks – **never throw on fetch failure**.

---

## Calculation Logic

### Girokonto (0% interest)
Real value decays purely due to inflation each year:
```
realValue[y] = realValue[y-1] / (1 + inflationRate[y])
```

### Tagesgeld (ECB overnight rate)
Nominal grows by the interest rate, then deflated:
```
tagesgeldNominal[y] = tagesgeldNominal[y-1] * (1 + interestRate)
realValue[y]        = realValue[y-1] * (1 + interestRate) / (1 + inflationRate[y])
```

### Historic inflation mode
When enabled, `getInflationRate()` cycles through the selected Eurostat year range using modular indexing:
```js
historicRates[(yearIdx - 1) % historicRates.length] / 100
```
This means a projection longer than the selected historic range will **repeat the pattern**.

---

## UI Conventions

- **Language:** All copy is in German. Keep new text in German.
- **Color palette:**
  - Dark navy `#02193d` – primary text and Girokonto chart line
  - Blue `#3096ff` – accent, Tagesgeld, interactive elements
  - Red `#e04545` – loss/negative stats
  - Green `#1d8a4e` – retained value stats
  - Muted `#5a6b87` / `#9aa5b4` – secondary labels
  - Background `#f7f5f3`; cards `#fff`
- **Transitions:** `0.25s cubic-bezier(0.4, 0, 0.2, 1)` is the standard easing used throughout
- **Border radius:** Cards use `32px`; inner elements use `16px`–`24px`
- **Naming:** CSS classes use `kebab-case`; JS variables/functions use `camelCase`; JS constants use `SCREAMING_SNAKE_CASE`
- **Number formatting:** Always use `fmtEuro()` or `fmtK()` for displayed euro values (ensures German locale thousands separators)

---

## Slider Ranges (UI Constraints)

| Slider | Min | Max | Step | Default |
|--------|-----|-----|------|---------|
| Savings (Erspartes) | €1,000 | €500,000 | €1,000 | €10,000 |
| Inflation rate | 0.5% | 15% | 0.1% | 5% |
| Time horizon (Jahre) | 1 | 50 | 1 | 10 |
| Year from (Historisch) | 2000 | 2025 | 1 | 2015 |
| Year to (Historisch) | 2000 | 2025 | 1 | 2025 |

---

## Development Workflow

Since this is a no-build, single-file project:

1. **Edit** `index.html` directly.
2. **Test** by opening `index.html` in a browser (no server needed for most features).
3. **CORS note:** The Eurostat and ECB API calls will be blocked if opened from `file://` in some browsers. Use a simple local HTTP server if you need to test live API data:
   ```bash
   python3 -m http.server 8080
   # then open http://localhost:8080
   ```
4. **No linting or formatting tools** are configured. Follow the existing style manually.

---

## Key Conventions to Follow

- **Single-file principle:** Keep all HTML, CSS, and JS in `index.html`. Do not split into separate files unless explicitly asked.
- **No framework introductions:** Do not add React, Vue, jQuery, or any other library without explicit approval.
- **No build step:** Do not introduce Webpack, Vite, or similar tooling without explicit approval.
- **Error resilience:** All external API calls must have a `try/catch` with a sensible fallback value and a silent `console.error`. Never let a fetch failure break the UI.
- **Re-render pattern:** The app uses a single `update()` function that recalculates and re-renders everything. When adding features, integrate with this pattern – avoid ad-hoc DOM mutations outside of `update()`.
- **Chart teardown:** `chart.destroy()` is called before creating a new Chart.js instance to avoid canvas memory leaks. Maintain this pattern.
- **German locale numbers:** Always use `toLocaleString('de-DE')` or the existing `fmtEuro()`/`fmtK()` helpers when displaying monetary values.
- **No persistent state:** The app intentionally resets on page reload. Do not introduce `localStorage` or cookies unless explicitly asked.
