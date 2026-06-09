# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A single-page static web app that reads `data.xlsx` and renders it as a searchable, paginated card grid with a detail view for each record. The project is titled "Initial Deposit for Silver Jubilee" and tracks deposit records by name, branch, and amount.

## Deployment

The app is deployed automatically to GitHub Pages via `.github/workflows/static.yml` on every push to `main`. The entire repository root is served as static content — no build step required.

To test locally, you must serve the files over HTTP (not `file://`) because the app fetches `data.xlsx` via `fetch()`:

```sh
python3 -m http.server 8080
# then open http://localhost:8080
```

Opening `index.html` directly in a browser will trigger the file-picker fallback instead of auto-loading `data.xlsx`.

## Architecture

Everything lives in `index.html` — styles, markup, and JavaScript are all inline. There are no external dependencies beyond the CDN-loaded SheetJS library (`xlsx.full.min.js`).

**Data flow:**
1. On page load, `fetch('data.xlsx')` attempts to auto-load the spreadsheet.
2. On failure, a manual file-picker UI is shown as a fallback.
3. SheetJS parses the workbook; the first sheet is always used.
4. `parseWorkbook()` trims all keys and string values and returns an array of plain objects.
5. `initData()` sets `allRows` and `filtered`, then calls `renderList()`.

**Column resolution** — the JS looks up columns by normalized (lowercased, trimmed) header name:
- `NAME_COL = 'name'` → card title and detail page heading
- `BRANCH_COL = 'branch'` → colored badge on cards and subtitle on detail page
- `AMOUNT_COL = 'initial deposit amount paid'` → card subtext
- `TOTAL_COL = 'total'` → shown in the summary box at the bottom of the detail view
- `OUTSTANDING_COL = 'outstanding'` → shown alongside Total in the summary box
- `CURRENCY_COLS = { AMOUNT_COL, TOTAL_COL, OUTSTANDING_COL }` → any column in this set gets a `₹` prefix via `formatCurrency()` when rendered
- `SKIP_COLS = { 'transaction screenshot', 'timestamp' }` → hidden from the detail field list

If the expected column names are not found, the code falls back to positional headers (`headers[0]`, `headers[1]`). `formatCurrency()` is idempotent — it only prepends `₹` if the value doesn't already start with it, so values stored as strings in the xlsx won't be double-prefixed.

**Views** — three mutually exclusive `display` states controlled by `showView(view)`:
- `#loader` — shown while `data.xlsx` is fetching
- `#file-picker` — shown on fetch failure
- `#list-view` — paginated card grid with search
- `#detail-view` — full record fields for a clicked card, with a two-cell summary box (`#detail-summary`) pinned below the field list showing Total and Outstanding

The Total and Outstanding columns are intentionally excluded from the generic `#detail-fields` list and only appear in the summary box.

**Search** filters `allRows` against all column values (case-insensitive substring match) and resets to page 1. Cards use `align-items: start` / `align-content: start` on the grid so they don't stretch to fill each other's height when results are sparse.

**Pagination** shows 10 items per page (`ITEMS_PER_PAGE = 10`) with a smart ellipsis algorithm in `paginationPages()`.

## Modifying the Data Schema

When the spreadsheet columns change:
- Update `NAME_COL`, `BRANCH_COL`, `AMOUNT_COL`, `TOTAL_COL`, `OUTSTANDING_COL` constants to match the new header names (lowercase).
- Add or remove entries from `CURRENCY_COLS` to control which columns get the `₹` prefix.
- Add any columns that should be hidden from the detail field list to `SKIP_COLS`.
- Column lookup is case-insensitive and whitespace-tolerant, so minor header variations are handled automatically.
