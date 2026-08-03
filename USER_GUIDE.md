# CAT 2026 — Mock Performance Tracker

A formula-driven analytics workbook for tracking **CAT** mock-test performance. You enter raw results in **one** place; every metric, chart, and breakdown derives itself and updates automatically.

> **Core principle:** *Enter once, derive everywhere.* If a number can be calculated, it is never typed. Nothing downstream can go stale or disagree with the source.

---

## Table of Contents

1. [What this workbook is](#1-what-this-workbook-is)
2. [Quick start](#2-quick-start--where-to-enter-data)
3. [Sheet map](#3-sheet-map)
4. [Mock Database — column by column](#4-mock-database--column-by-column)
5. [Data validation — what it will refuse](#5-data-validation--what-it-will-refuse)
6. [Reading the Dashboard](#6-reading-the-dashboard)
7. [The charts](#7-the-charts)
8. [How to evaluate what you see](#8-how-to-evaluate-what-you-see)
9. [Targets, trends & projections](#9-targets-trends--projections)
10. [Features at a glance](#10-features-at-a-glance)
11. [**FAQ** & gotchas](#11-faq--gotchas)

---

## 1. What this workbook is

A single-source-of-truth tracker. You log raw mock results in the **Mock Database** sheet. Everything else — 15+ metrics per section, 9 charts, provider/series breakdowns, targets, projections, and diagnostics — is computed by formula and refreshes itself as you add mocks.

Three ideas run through the whole design:

| Idea | What it means for you |
|---|---|
| **Enter once, derive everywhere** | You never type a number that could be calculated. One entry updates the entire workbook. |
| **Blank-safe** | A blank means **no data**, never zero. A mock with no percentile is *excluded* from percentile stats, not counted as 0. |
| **Self-sizing** | Charts, tables, and statistics size themselves to however many mocks you've completed — no manual range editing, ever. |

---

## 2. Quick start — where to enter data

**The only sheet you type in is `Mock Database`.** Every other sheet is locked output.

| To do this… | Do this |
|---|---|
| **Log a completed mock** | Find its row → enter **Actual Date**, then section **Scores, Percentiles, Attempts, Correct**. Status flips to *Completed* automatically and all analytics update. |
| **Plan a future mock** | Enter **Test Name** + **Planned Date** (+ Provider / Series / Mock Type). It counts as *Planned* but stays *Pending* until an Actual Date is entered. |
| **Change a target** | `Calculations` sheet, cells **B38:B41** (blue-on-yellow = editable). |
| **Add a dropdown option** | `Lists` sheet — add to the relevant column; dropdowns pick it up automatically. |

> 💡 **Leave blank what you don't have.** Past **CAT** papers have no percentile — leave those cells empty. The workbook handles the gap gracefully.

---

## 3. Sheet map

| Sheet | Purpose | Editable? |
|---|---|---|
| **Dashboard** | Headline view: KPI cards, 4 section panels, insight strips, 9 charts, Mock Index (cols R:AB) | 🔒 Locked |
| **Mock Database** | **THE DATA** — your raw entry sheet, 150 pre-created rows (3–152) | ✏️ Inputs only |
| **Section Analysis** | Deep per-section metrics + Section Summary + Attempt Efficiency table | 🔒 Locked |
| **Provider Analysis** | Performance split by Provider, Series, and Mock Type | 🔒 Locked |
| **Calculations** | The engine — all metrics, targets, trends, projections | 🔒 Locked except targets B38:B41 |
| **Chart_Data** | Internal chart plumbing — **never edit** | 🔒 Locked |
| **Lists** | Dropdown source lists | ✏️ Editable |

---

## 4. Mock Database — column by column

Header row is **row 2**; data rows are **3–**152****. Input columns are unlocked; every calculated column is locked so you can't overwrite a formula by accident.

| Col | Field | Source | Notes |
|---|---|---|---|
| A | Mock No | **You** | Whole number 1–400. Tie-breaker when two mocks share an Actual Date. |
| B | Planned Order | *Auto* | Rank by Planned Date. |
| C | **Attempt Order** | *Auto* | Rank by Actual Date among completed mocks — 1, 2, 3… no gaps. **The join key the whole engine runs on**, and the number on every chart's x-axis. |
| D | Test Name | **You** | e.g. *SimCAT 9*. |
| E / F / G | Provider / Series / Mock Type | **You** | Dropdowns sourced from `Lists`. |
| H | Planned Date | **You** | A row is *Planned* once it has a Test Name + Planned Date. |
| I | **Actual Date** | **You** | **The switch** — entering it flips Status to *Completed* and pulls the mock into every stat and chart. |
| J | Status | *Auto* | `Completed` if Actual Date filled, else `Pending`. |
| K | Analysis Done | **You** | Yes/No. Drives the Analysis Rate KPI. |
| L | Notes | **You** | Free text. |
| M–R | **VARC** block | mixed | Score, %ile, Attempts, Correct = **you**. Incorrect (= Att − Correct) and Accuracy (= Correct ÷ Att) = *auto*. |
| S–X | **DILR** block | mixed | Same pattern. |
| Y–AD | **QA** block | mixed | Same pattern. |
| AE–AJ | **Overall** block | mixed | Score (auto = sum of 3 sections), **%ile (you — cannot be derived)**, Attempts/Correct/Incorrect (auto sums), Accuracy (auto). |

---

## 5. Data validation — what it will refuse

| Field | Rule |
|---|---|
| Dates (Planned / Actual) | A real date, 2020–2035 |
| Section scores | Whole number, −100 to 200 (negatives allowed — CAT has negative marking) |
| Percentiles | 0–100, decimals allowed |
| Attempts | Whole number, 0–150 |
| **Correct** | Whole number ≥ 0, **and never more than that section's Attempts** |
| Provider / Series / Mock Type / Analysis | Must come from the dropdown |

> The Correct ≤ Attempts guard blocks the silent error that used to produce negative *Incorrect* and >**100**% accuracy.

---

## 6. Reading the Dashboard

- ****KPI** cards (top):** Days to **CAT** · Completed/Planned · Completion Rate · Analysed · Last Attempted.
- **Four section panels** (Overall, **VARC**, **DILR**, QA), each showing: Latest, Average, Median, Highest, Lowest, Std Dev, Roll Avg 5, Roll Avg 10, Improvement, **Trend/mock**, Consistency — as **Score** and **%ile** columns, with ▲/▼ direction arrows.
- **Insight strips:** auto-generated — strongest/weakest/most-improved section, projected next score, trend, and target readiness.
- **Mock Index (cols R:AB):** a live legend mapping each chart's attempt-number back to a named mock, with every graphed metric. Grows automatically.

The **Latest** row shows the single most recent mock. If that mock has a blank metric (e.g. a Past-**CAT** paper's percentile), the Latest cell is **blank** — it never shows a stale older value.

---

## 7. The charts

| # | Chart | Reads |
|---|---|---|
| 1 | Overall Score trend | Overall score by attempt |
| 2 | Overall Percentile trend | Overall %ile (zoomed 75–100) |
| 3 | Section Scores | VARC/DILR/QA scores |
| 4 | Section Percentiles | VARC/DILR/QA %iles (zoomed 20–100) |
| 5–7 | VARC / DILR / QA detail | Score (left axis) + %ile (right axis, 0–100) |
| 8 | All-sections comparison | 3 section scores + **Overall** line on top |
| — | **Sectional Balance (radar)** | Average %ile vs Best %ile per section |

All trend charts are **XY-scatter** so the x-axis is attempt number and auto-extends as you log mocks. Missing percentiles show as **line breaks**, not zero-drops.

---

## 8. How to evaluate what you see

| Signal | How to read it |
|---|---|
| **Trend / mock** | Positive = improving. The size is points gained per mock. A flat trend on a high percentile = plateau. |
| **Consistency Index** | 100 × (1 − StdDev/Avg). Higher = steadier. Low consistency with a high average = volatile; stabilise before CAT. |
| **Roll Avg 5 vs Average** | Recent form vs lifetime. Roll-5 above lifetime = you're on an upswing. |
| **Radar gap (Avg vs Best)** | Big gap on a section = untapped headroom; you've hit that level before and can again. |
| **Attempt Efficiency (marks/attempt)** | Low value + many attempts + many wrong = over-attempting into negative marking. |
| **Colour bands** | 7-tier: personal-best → excellent (≥+1 SD) → above median → typical → below median → poor (≤−1 SD) → worst. Recomputes live. |

---

## 9. Targets, trends & projections

Set on `Calculations` ****B38**:**B41**** (blue-on-yellow inputs):

| Target | Default |
|---|---|
| Overall %ile | 99.9 |
| VARC Score | 55 |
| DILR Score | 40 |
| QA Score | 45 |

For each, the workbook shows **Recent (5-mock avg)**, **Gap to target**, a **Status verdict** (✓ On target / Closing +X/mock / Behind), and **Mocks @ trend** (how many more mocks at your current rate to hit the target). A *not closing* verdict means that section's trend is flat — more volume alone won't get you there.

---

## 10. Features at a glance

- ✅ Single-entry data model — type once, derive everywhere
- ✅ Blank-safe statistics (no false zeros)
- ✅ Self-sizing charts, tables, and metrics
- ✅ 7-band dynamic conditional formatting (mean/median/SD based)
- ✅ Trend, projection, and coefficient-of-variation metrics
- ✅ Target tracking with pacing and mocks-to-target reality check
- ✅ Attempt-efficiency diagnostics (marks per attempt, negatives)
- ✅ Provider / Series / Mock-Type breakdowns
- ✅ Sectional-balance radar
- ✅ Live Mock Index tying chart points to named mocks
- ✅ Full input validation + locked formulas
- ✅ Excel **and** Google Sheets compatible formulas

---

## 11. FAQ & gotchas

**Why does the x-axis show numbers, not mock names?** The charts use a value (scatter) axis so they can auto-grow forever. The **Mock Index (R:AB)** maps each number to its named mock.

**A percentile cell is blank — is that a bug?** No. Past-**CAT** papers have no percentile; leave it blank and the workbook excludes it correctly.

**I added a mock but a chart didn't move.** It will after a recalc. If not, press `Ctrl+Alt+F9` to force a full recalc.

**The workbook won't let me type in a cell.** That cell is a locked formula. Only Mock Database inputs, Calculations **B38**:**B41**, and Lists are editable. To unlock everything: *Review → Unprotect Sheet* (no password).

**Requirements:** the auto-growing charts and spill-based tables need **Excel **365** / **2021**+** or **Google Sheets** (both support the `**FILTER**` function). Older Excel will show the data but not auto-expand.
