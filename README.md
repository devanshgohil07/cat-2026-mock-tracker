# CAT 2026 — Mock Performance Tracker

A formula-driven Excel workbook that turns raw CAT mock results into a self-updating analytics dashboard. Enter each mock once; every metric, chart, target, and breakdown updates itself.

[![License: CC BY-NC 4.0](https://img.shields.io/badge/License-CC%20BY--NC%204.0-blue.svg)](https://creativecommons.org/licenses/by-nc/4.0/)
![Version](https://img.shields.io/badge/version-1.2.0-green.svg)
![Excel](https://img.shields.io/badge/Excel-365%20%7C%202021%2B-217346.svg)

## Highlights

- **Enter once, derive everywhere** — 15+ metrics per section, all auto-computed
- **Auto-growing charts** — x-axis extends itself as you log mocks (no range editing)
- **7-band dynamic conditional formatting** (mean / median / SD based)
- **Targets, trends, projections & pacing** with a mocks-to-target reality check
- **GP (exponentially) weighted average** — geometric decay across your full history
- **Planned-mocks-by-month schedule table** — self-sizing from your planned dates
- **Attempt-efficiency diagnostics**, provider/series breakdowns, sectional-balance radar
- **Validated inputs + locked formulas** so nothing breaks by accident

## Getting started

1. Download `CAT_Analytics_Dashboard.xlsx`.
2. Open in **Excel 365 / 2021+** (recommended) or **Google Sheets** — both support the `FILTER` function the auto-growth depends on.
3. Enter mocks in the **Mock Database** sheet only. Everything else is locked output.

## Sheet map

| Sheet | Purpose | Editable? |
|---|---|---|
| **Dashboard** | KPI cards, 4 section panels, insight strips, schedule table, 9 charts, Mock Index | 🔒 Locked |
| **Mock Database** | **THE DATA** — your raw entry sheet, 150 pre-created rows | ✏️ Inputs only |
| **Section Analysis** | Deep per-section metrics + Attempt Efficiency | 🔒 Locked |
| **Provider Analysis** | Performance split by Provider, Series and Mock Type | 🔒 Locked |
| **Calculations** | The metric engine, targets, trends, projections | 🔒 Except B38:B41, B42 |
| **Chart_Data** | Internal chart plumbing — never edit | 🔒 Locked |
| **Lists** | Dropdown source lists | ✏️ Editable |

## What's new in 1.2.0

- **Fixed:** rolling averages on all percentile columns were anchored to a per-column
  `COUNT`, which is lower than the mock count whenever a mock has no percentile
  (Past CAT papers). Windows landed several mocks too early and understated recent
  form — the overall gap-to-target was overstated by ~8.5 percentile points.
- **Added:** *Planned Mocks by Month* table on the Dashboard (`R8:V19`) — planned /
  done / pending / % done, self-sizing from your earliest to latest planned date.
- **Added:** *GP Weighted Average* — geometric decay `r` over the full score history,
  with `r` as a live input.
- **Changed:** Series list compacted; the **Others** row is now a true catch-all that
  captures any series not explicitly listed.
- **Docs:** attempt-efficiency guidance corrected (see below).

## Reading the numbers

CAT marks **+3 correct, −1 incorrect**, so the break-even accuracy for attempting a
question is **25%**. If your section accuracy is comfortably above that, leaving
questions unattempted costs you marks. Use *Marks / Attempt* in the Attempt
Efficiency table alongside your attempt count — a high marks-per-attempt figure
paired with a low attempt count means unused headroom, not efficiency.

## Requirements & limitations

- Auto-growth needs `FILTER` (Excel 365 / 2021+ or Google Sheets). Older Excel shows the data but won't self-expand.
- Input ranges cover 150 mock rows. Beyond that, extend the `3:152` ranges (see `docs/TECH_SPEC.md` §11).
- Percentile charts use a fixed y-axis floor for readability — see `docs/TECH_SPEC.md` §7 for the trade-off.

## Documentation

📘 Full user guide: [`docs/USER_GUIDE.md`](docs/USER_GUIDE.md)
🔧 Technical spec & build history: [`docs/TECH_SPEC.md`](docs/TECH_SPEC.md)

## License

Released under [CC BY-NC 4.0](LICENSE) — free to use and adapt **with attribution**, non-commercially.
Created by **Devansh Gohil**. If you use or adapt it, please keep the credit.
