# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A personal NSE (Indian stock market) trading toolkit. The **Rocket Scanner** is a standalone single-file HTML/JS web app for scanning and ranking NSE stocks. Runs entirely client-side; no build step, no server, no dependencies to install.

## Running the Scanner

Open `Rocket_Scanner.html` directly in a browser (double-click or drag in). Upload files from `Scanner Uploads/`:
- `ALL NSE.csv` — full NSE universe (primary input, usually a TradingView screener export)
- `Holdings.csv` / `Positions.csv` — Zerodha portfolio exports
- `TRADEBOOK.csv` — Zerodha trade history
- NSE ZIP files — bhav copy, 52-week high/low, surveillance, bulk/block deals

`Rocket_Scanner - Last Working.html` is a backup checkpoint of a previously stable version.

## Scanner Architecture

`Rocket_Scanner.html` is a ~3000-line single-file app with all CSS, HTML, and JS inlined. No build step — edit and reload.

### Data Flow

1. **File ingestion** → `detectNSE()` auto-routes uploaded files to the right parser (`parseBhavdata`, `parse52W`, `parseSurv`, `parseDeal`, `parseCSV`). NSE enrichment data lands in module-level maps: `NSE_BHAV`, `NSE_52W`, `NSE_SURV`, `NSE_BULK`, `NSE_BLOCK`.

2. **Engine** → `runEngine(raw, sessionTag)` (line ~688) is the core scoring pipeline:
   - Auto-detects columns from CSV headers using heuristics (no hardcoded column names)
   - Computes derived features: `delivery_pct`, `range_pos`, `pct_from_52w_high`, `peak_retention`, `sector_breadth`, `sector_rel_strength`
   - **Hard filters**: removes upper-circuit stocks (≥19.5% change), surveillance-listed, illiquid (vol <5000 or shareholders <500), missing Piotroski score, missing ATR, delivery <30%, or fading candle (peak retention <50%)
   - **Market regime detection**: classifies session as `bull`/`bear`/`neutral` based on advancing breadth (>55% = bull, <35% = bear)
   - **mRMR scoring**: computes Pearson correlations of each feature against the "rocket" target (stocks up ≥X% today), applies Minimum Redundancy Maximum Relevance weighting. Regime-conditional accumulators (`ACC_CORR_BULL/BEAR/NEUT`) accumulate correlations across sessions — the longer the history, the more stable the feature weights.
   - **Quality-adjusted target**: yesterday's rockets that held today are included in the positive set; yesterday's rockets that collapsed are excluded.

3. **Persistence** → All state is stored in `localStorage` (keys prefixed `rs_`). `Rocket_Brain.json` is an exportable/importable brain dump of all localStorage keys for sharing state across browsers.

4. **Rendering** → `renderStats()`, `renderTable()`, `renderMethodology()`, `renderHead()` build the UI. Table is paginated; `applyFilters()` and `applySort()` operate on the `ENGINE_DATA` result.

5. **Basket output** → Selected stocks generate `Zerodha_Basket_Buy.json` / `Zerodha_Basket_Sell.json` using `Basket_Template.json` as the schema.

### Key Global State

| Variable | Purpose |
|---|---|
| `ENGINE_DATA` | Output of `runEngine()` — weights, scores, feature metadata |
| `ACC_CORR` / `ACC_CORR_BULL/BEAR/NEUT` | Accumulated cross-session Pearson correlations per regime |
| `PREV_SNAP` | Previous session's full feature vectors (symbol → features), used for lagged correlation |
| `CURRENT_REGIME` | `bull` / `bear` / `neutral` for current session |
| `NSE_BHAV/52W/SURV` | NSE enrichment lookups keyed by symbol |
| `HOLDINGS` / `POSITIONS` | Zerodha portfolio state |

### Key localStorage Keys

`rs_filters`, `rs_corr`, `rs_corr_bull`, `rs_corr_bear`, `rs_corr_neut`, `rs_snap`, `rs_snap_prev`, `rs_all`, `rs_meth`, `rs_hold`, `rs_pos`, `rs_tradebook`, `rscanner_ver`

### Version System

`CODE_HASH` is computed from the inline `<script>` content on each load. If it differs from the stored hash, the version counter auto-increments. This is how the title `NSE Rocket Scanner vN` updates.

## Key Files

- `Rocket_Brain.json` — Exportable brain dump of all `localStorage` state; import/export via the UI to transfer accumulated session history across browsers
- `Basket_Template.json` — Schema reference for Zerodha basket order JSON format
- `Scanner Uploads/` — Sample input files (not committed as live data)
