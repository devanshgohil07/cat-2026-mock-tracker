# CAT 2026 Mock Tracker — Technical Specification & Build History

**Version:** 1.2.0
**Audience:** an engineer, AI assistant, or maintainer who needs to understand this workbook end-to-end — philosophy, architecture, every formula, and the reasoning behind each design decision, including the dead-ends that were rejected and why.

---

## 0. Philosophy

The workbook exists to answer one question continuously: *"Am I on track for my CAT target, and where do I fix things?"* Every design choice serves three non-negotiables:

1. **Auditability** — a human can click any number and trace it to source. No value is computed off-sheet and pasted.
2. **Single source of truth** — raw data lives in exactly one place (`Mock Database`). Everything else is a pure function of it.
3. **Zero-maintenance growth** — logging a mock must update *everything* (stats, charts, tables) with no manual range edits.

A recurring tension throughout the build: **Excel-and-Sheets portability** vs. **Excel-only power**. Rule adopted: all *formulas* stay portable (no `LET`, no `@`, no spill-only operators in the engine); Excel-specific mechanics are confined to the chart layer, which doesn't port to Sheets anyway.

---

## 1. Requirements (as they emerged)

| # | Requirement | Origin |
|---|---|---|
| R1 | Track mock scores + percentiles across VARC/DILR/QA/Overall | Initial |
| R2 | Rich per-section stats (avg, median, extremes, variance, rolling, trend, consistency) | Initial |
| R3 | Charts for score & percentile trends | Initial |
| R4 | Provider / Series / Mock-Type breakdowns | Initial |
| R5 | Blank-safe: never treat a missing value as 0 | Audit finding |
| R6 | Charts and stats must auto-grow as mocks are added | Repeated user pain |
| R7 | Dynamic, statistically-grounded conditional formatting | User |
| R8 | *Latest* must reflect one mock, blanks stay blank | User |
| R9 | Targets (99.9%ile overall; VARC 55 / DILR 40 / QA 45 scores) | User |
| R10 | 7-band conditional formatting | User |
| R11 | Genuine extra insights where scope exists | User |
| R12 | True dynamic chart x-axis | User (critical) |
| R13 | Point-to-mock traceability | User |
| R14 | Visual coherence across all added tables | User |
| R15 | Input validation + lock all non-input cells | User |
| **R16** | **Schedule visibility — planned mocks grouped by month** | **v1.2.0** |
| **R17** | **Full-history weighted average with geometric decay** | **v1.2.0** |

---

## 2. Architecture

```
Mock Database  (raw inputs, rows 3-152)
      │  (Attempt Order = join key)
      ▼
Chart_Data  (ledger + mirrors + spill sources)   ← internal plumbing
      │
      ├──► Calculations  (metric engine: B:I grid × metric rows, + targets/trends)
      │          │
      │          ├──► Dashboard  (cards, panels, insight strips, month table)
      │          ├──► Section Analysis
      │          └──► Provider Analysis
      │
      └──► Charts (9)  + Mock Index (Dashboard R25 down)
```

### 2.1 The ledger (`Chart_Data`)
A **compacted, packed** copy of completed mocks, one row per mock, no interior gaps, indexed by Attempt Order.

- **Col A** — index 1…400 (always populated; structural).
- **Cols B–K** — the ledger proper, pulled by `INDEX`/`MATCH` on Attempt Order.
  Blank source cells are guarded to return `""` (not `0`) via an `ISBLANK(OFFSET(...))` test.
  Column map: B=Mock name(D), C=Date(I), D=VARC Sc(M), E=VARC %(N), F=DILR Sc(S), G=DILR %(T), H=QA Sc(Y), I=QA %(Z), J=Ovr Sc(AE), K=Ovr %(AF).
- **Cols O–V — NA-mirror.** Same values but empties → `NA()`: `=IF($D2="",NA(),$D2)`. Used by charts so a missing percentile becomes a **line gap**, not a zero.
- **Cols Z–AH — `FILTER` spills** (the dynamic engine): `=FILTER(<mirror col>2:401, $B$2:$B$401<>"", "")` → exactly N rows, auto-resizing.
- **Cols AJ–AL — radar source** (section × Avg%ile, Best%ile).
- **Cols AM/AN — scatter X source:** `AM = IF($B2="",NA(),$A2)` (NA-guarded attempt index).

> ⚠️ **Ledger blanks are empty strings, not true blanks.** They are produced by formulas, so `""` is *text*. Any arithmetic that touches the range directly (`range * weight`) throws `#VALUE!`. Aggregations that skip text (`AVERAGE`, `MEDIAN`, `STDEV`, `SLOPE`, `COUNT`) are unaffected; array arithmetic must coerce first — see §3.2.

### 2.2 The engine (`Calculations`)
A grid of **metric rows × 8 metric columns**. Column→ledger map: B→J, C→K, D→D, E→E, F→F, G→G, H→H, I→I.

Every statistic reads a **packed, blank-safe** ledger column, so no `>0` filters are needed and native functions skip blanks correctly.

---

## 3. Formula reference (the engine)

Let `R` = the ledger column range (e.g. `Chart_Data!$J$2:$J$401`), `O` = its header cell, `n` = **`$B$13`** (the mock count).

| Metric (row) | Formula | Notes |
|---|---|---|
| Latest (2) | `=IF($K$2="","",IF(INDEX(R,$K$2)="","",INDEX(R,$K$2)))` | Anchored to one mock via `K2`. Blank stays blank. |
| Average (3) | `=IFERROR(AVERAGE(R),"")` | Skips blanks natively. |
| Median (4) | `=IFERROR(MEDIAN(R),"")` | |
| Highest (5) | `=IFERROR(MAX(R),"")` | |
| Lowest (6) | `=IFERROR(MIN(R),"")` | |
| Variance (7) | `=IFERROR(VAR(R),"")` | |
| Std Dev (8) | `=IFERROR(STDEV(R),"")` | |
| Roll Avg 5 (9) | `=IFERROR(AVERAGE(OFFSET(O,MAX(1,$B$13-4),0,MIN(5,$B$13),1)),"")` | **Window anchored to the mock count — see §5.** |
| Roll Avg 10 (10) | `=IFERROR(AVERAGE(OFFSET(O,MAX(1,$B$13-9),0,MIN(10,$B$13),1)),"")` | |
| Improvement (11) | `=IFERROR(Latest - INDEX(R,1),"")` | vs first mock. |
| Best-till-date (12) | `=IFERROR(MAX(R),"")` | |
| Count (13) | `=COUNT(R)` | Per-column count of *values*. Informational only. |
| Last-5-vs-Avg (14) | `=IFERROR(RollAvg5 - Average,"")` | |
| Last-10-vs-Avg (15) | `=IFERROR(RollAvg10 - Average,"")` | |
| Consistency (16) | `=IFERROR(100*(1-StdDev/Average),"")` | Higher = steadier. |
| AvgDaysBetween (17) | `MAXIFS(date)-MINIFS(date)` ÷ `MAX(n-1,1)` | Compare to PaceNeeded (31). |
| **Trend (32)** | `=IFERROR(SLOPE(R, Chart_Data!$A$2:$A$401),"")` | Points gained per mock. |
| **Projected next (33)** | `=IFERROR(FORECAST($B$13+1, R, Chart_Data!$A$2:$A$401),"")` | Linear extrapolation to the next attempt. |
| **Coeff. of Variation (34)** | `=IFERROR(100*StdDev/Average,"")` | Volatility, scale-free. |
| **GP Weighted Avg (35)** | see §3.2 | Overall score only. |

**Latest anchor** `K2 = =IFERROR(COUNT(Chart_Data!$J$2:$J$401),"")` — because the ledger is packed, the count of the *score* column equals the row index of the newest mock. All *Latest* cells `INDEX` into that one row (R8).

### 3.1 Rolling-window anchoring — critical

Windows are keyed to **`$B$13`** (count of the Overall *score* column = the true mock count), **never** to the column's own row-13 count.

The ledger is packed **by mock**, not by value. A percentile column has interior gaps wherever a Past-CAT paper sits, so `COUNT` on a percentile column is lower than the mock count and cannot be used as a row index. Using it walks the `OFFSET` window backwards by exactly the number of missing percentiles.

Semantics adopted: **Roll-N = the last N mocks, blanks skipped.** The window is always the N most recent attempts; `AVERAGE` ignores the ones without a value. This keeps score and percentile windows directly comparable. (The rejected alternative — "the last N mocks that *have* a value" — reaches further back in time and desynchronises the two columns.)

### 3.2 GP weighted average (R17)

Geometric decay across the full history. With attempts `i = 1…n` (i = n is latest), value `xᵢ`, ratio `r`:

```
        Σ δᵢ · r^(n−i) · xᵢ
S_n  =  ───────────────────         δᵢ = 1 if xᵢ present, else 0
          Σ δᵢ · r^(n−i)
```

The denominator is a geometric series `(1−rⁿ)/(1−r)`; the recursive form is standard EWMA with `α = 1−r`, including the finite-sample correction `(1−rⁿ)` that matters at low n.

Implementation (`Calculations!B35`, Overall score only):

```excel
=IFERROR(
   SUMPRODUCT(IF(cd_Ovrsc="",0,cd_Ovrsc)*($B$42^($K$2-cd_Idx)))
 / SUMPRODUCT(IF(cd_Ovrsc="",0,1)      *($B$42^($K$2-cd_Idx))),
 "")
```

- `$B$42` — the decay ratio, a **live input** (default 0.945).
- `cd_Idx` — new named range `=OFFSET(Chart_Data!$A$2,0,0,COUNT(Chart_Data!$J$2:$J$401),1)`, the attempt index sized to the live mock count.
- `IF(range="",0,range)` is **required**, not defensive: ledger blanks are empty strings and would otherwise throw `#VALUE!`.
- Both ranges are `OFFSET`-sized, so the exponent `$K$2 − cd_Idx` recomputes off the live count. Never bind this to a padded range — `r^(negative)` on trailing empty rows overflows to `#NUM!` before the zero-multiply can suppress it.

Reference values for `r`:

| r | α | Half-life | Centre of mass |
|---|---|---|---|
| 0.80 | 0.200 | 3.1 | 4.0 |
| 0.87 | 0.130 | 5.0 | 6.7 |
| 0.90 | 0.100 | 6.6 | 9.0 |
| 0.945 | 0.055 | 12.3 | 17.2 |

### 3.3 Progress block (rows 22–31)
- PlannedTotal `=COUNTIFS(D<>"", H<>"")`; Completed `=COUNTIF(J,"Completed")`; Pending = diff.
- CompletionPct, AnalysisDone/Pending/Pct analogous.
- DaysToCAT `=DATE(2026,11,29)-TODAY()` — CAT 2026 is Sunday 29 Nov 2026 (IIM Indore).
- **PaceNeeded (31)** `=DaysToCAT/Pending` — days available per remaining mock; compare to actual avg gap (B17).

> **Note:** this block uses ranges to row 400 while the rest of the workbook stops at 152. Harmless (rows 153+ are empty) and more future-proof, but inconsistent — validation and conditional formatting stop at 152.

### 3.4 Targets & readiness (rows 36–41)
Inputs **B38:B41**. Per row: Recent = RollAvg5; Gap = Target − Recent; Status verdict; Mocks@trend `=ROUND(Gap/Trend,0)`.

### 3.5 Attempt efficiency (rows 43–50)
`AVERAGEIFS(..., Status,"Completed")` over the Attempts/Correct/Incorrect/Accuracy columns. **Marks/Attempt** = avg score ÷ avg attempts. Auto-insight string flags most/least efficient section and most negatives.

> **Interpretation guidance (corrected in v1.2.0).** CAT marks +3/−1, so break-even accuracy is **25%**. Marks-per-attempt must be read *with* attempt count: a high figure on a low attempt count is unused headroom, not efficiency. Expected value of a marginal attempt ≈ `3a − (1−a)`.

### 3.6 Planned Mocks by Month (R16) — `Dashboard!R8:V19`
Self-generating month spine:

```excel
R10 =IFERROR(EOMONTH(MINIFS(PD, NAME,"<>", PD,">0"),-1)+1,"")
R11 =IF($R10="","",IF(EOMONTH($R10,0)+1 > MAXIFS(PD, NAME,"<>", PD,">0"),"",EOMONTH($R10,0)+1))   ' filled down
```

Counts per month use the canonical planned test (`Test Name` **and** `Planned Date` both non-empty):

```excel
S10 =IF($R10="","",COUNTIFS(NAME,"<>", PD,">="&$R10, PD,"<="&EOMONTH($R10,0)))
T10 = …same… & ST,"Completed"          ' Done
U10 =$S10-$T10                          ' Pending
V10 =$T10/$S10                          ' % Done
```

Nine month slots (rows 10–18) plus a Total row (19); surplus rows resolve to `""`.

> **Placement constraint:** this block sits in rows 8–19 because those rows are uniformly 16.5pt. Rows 3–7 carry the KPI-card geometry (37.5 / 12.8 / **4.5 spacer** / 21.8 / 12.8) and would clip any table placed there. See §10.

---

## 4. The dynamic-chart problem (R6/R12) — full history

**Symptom:** charts froze at the mock count they had when the add-in bound them.

**Root cause:** the Office.js API resolves any range or defined-name you bind to a **static address** and writes *that* into the chart's `SERIES` formula.

| Approach tried | Result | Why rejected |
|---|---|---|
| Bind to full padded range (`B2:B401`) | ❌ | A **category** axis reserves one slot per cell — empty, `""` or `#N/A` alike. 380 blank slots crushed the data into the left 5%. |
| Dynamic named range (`OFFSET`) in SERIES | ❌ | Office.js resolves the name to a static address at bind time. |
| `FILTER` spill + spill-ref binding (`Z2#`) | ❌ (false positive) | The SERIES stored the resolved address, not `Z2#`. |
| **XY-Scatter with value x-axis** | ✅ | A **value** axis auto-scales to plotted points and **skips `#N/A` pairs entirely**. Binding to the full 400-row range grows forever with no rebinding. |

**Final implementation:** all 8 trend charts are `XYScatterLines`. X = `Chart_Data!AM`; Y = the NA-mirror columns (O–V). **Verified end-to-end in v1.2.0:** injecting a 23rd mock extended every chart to 23 points, grew the ledger, spills, Mock Index and month table, and moved every statistic; removing it contracted all of them back to 22 with zero spill errors.

**Trade-off accepted (R13):** a value axis can't show text labels, so the x-axis shows attempt number. The **Mock Index** restores traceability.

**Office.js gotchas discovered:**
- Excel silently reverts an in-place line→scatter `chartType` change; each chart must be **deleted and re-added**.
- The enum must be the string `"XYScatterLines"` — `Excel.ChartType.xyScatterLines` is `undefined`.
- Chart operations must be done **one chart per `context.sync()`**.
- **`chart.chartType` returns `null` for dual-axis (combo) charts.** Charts 5–7 put score on the primary axis and percentile on the secondary, creating two plot groups, so there is no single type to report. The *series* still report `XYScatterLines`. **This is not a defect** — verified by rendering.
- **Axis bounds cannot be read to determine auto vs. manual.** `axis.maximum` returns a number either way. To detect a manual bound, build a probe chart on the same data on a scratch sheet and compare.
- **`range.format.rowHeight` is a row-level property.** Setting it on `R3:V17` resizes **rows 3–17 across the entire sheet**, not just those columns. Same for `columnWidth`. See §5.

---

## 5. Bugs found & fixed (audit trail)

| Bug | Impact | Fix |
|---|---|---|
| Variance/StdDev used broken `SUMPRODUCT`+`@` | Returned blank → Consistency blank | Rewrote to `VAR`/`STDEV` over packed ledger |
| `">0"` filter used as *not blank* | Silently deleted valid low/negative scores | Removed; rely on native blank-skipping |
| `MEDIAN(IF(@...))` | Non-portable implicit intersection | Packed ledger makes plain `MEDIAN` correct |
| Rolling-avg window keyed to wrong rows | Roll-5 averaged ~10 values | `OFFSET` keyed to the count cell |
| *Latest* via `MATCH(MAX(date))` | Wrong row on tied dates | Single anchor `K2` |
| Chart_Data was a static hand-pasted snapshot | Charts missing newest mock | Formula-driven packed ledger |
| CF: stacked rules, no `stopIfTrue` | White font on pale fill | Orthogonal bands, explicit priority |
| Provider Analysis labels hardcoded | Drift risk | Linked to `Lists` |
| Correct could exceed Attempts | Negative *Incorrect*, >100% accuracy | Validation guard |
| Blank ledger percentile → `INDEX` returned `0` | False zeros | `ISBLANK(OFFSET(...))` guard |
| **Percentile rolling windows anchored to per-column `COUNT`** | **Roll-5/10, Last-5/10-vs-Avg and `FORECAST` on all four percentile columns read a window shifted back by the number of missing percentiles. Overall Roll-5 %ile read 88.27 instead of 96.76; gap-to-target overstated by 8.5 points (244 mocks @ trend vs 66).** | **Anchor all windows to `$B$13`; `FORECAST` uses `$B$13+1`** |
| **Series list contained never-used entries** (AIMCAT, CDC, Rodha Mock) | Dead rows in Provider Analysis | Removed from `Lists`; rows deleted |
| **`Others` matched the literal label only** | Unlisted series counted nowhere | Rewritten as a residual excluding `$A$16:$A$20` by *cell reference* |
| **Row heights flattened by a styling call** | KPI cards and all four panels squashed to 15pt | Restored from untouched reference rows; month table relocated to rows 8–19 |

---

## 6. Conditional formatting (R7/R10)

**7-band diverging system**, applied to score cols (M,S,Y,AE), percentile cols (N,T,Z,AF) and accuracy cols (R,X,AD,AJ), rows 3–152. Priority = insertion order; all `stopIfTrue`; every rule guarded `<cell><>""`.

| Band | Condition | Fill / Font |
|---|---|---|
| 1 | `= MAX` | `#1B5E20` / white bold |
| 2 | `= MIN` | `#B71C1C` / white bold |
| 3 | `≥ AVERAGE + STDEV` | `#43A047` / white |
| 4 | `≤ AVERAGE − STDEV` | `#EF5350` / white |
| 5 | `> MEDIAN` | `#C8E6C9` / `#1B5E20` |
| 6 | `< MEDIAN` | `#FFE0B2` / `#E65100` |
| 7 | catch-all (= median) | `#FFF9C4` / `#616161` |

Verified present on row 152 (all 7 rules) — coverage reaches the last input row.

---

## 7. Charts inventory

| Chart | Type | X | Y series | Axis notes |
|---|---|---|---|---|
| 1 Overall Score | XYScatterLines | AM | U | y auto |
| 2 Overall %ile | XYScatterLines | AM | V | **y fixed 75–100** |
| 3 Section Scores | XYScatterLines | AM | O,Q,S | y auto |
| 4 Section %iles | XYScatterLines | AM | P,R,T | **y fixed 20–100** |
| 5–7 VARC/DILR/QA | *combo (reports `null`)* | AM | score (primary) + %ile (secondary 0–100, dashed) | dual axis |
| 8 All sections | XYScatterLines | AM | O,Q,S + U (Overall, thick) | y auto |
| Radar | RadarMarkers | section | Avg%ile, Best%ile | radial 0–100 |

Palette: VARC `#2E7D32`, DILR `#1565C0`, QA `#C62828`, Overall `#4527A0`, Percentile `#EF6C00`.

**Known limitation — fixed percentile floors.** Charts 2 and 4 clip any point below 75 / 20. Office.js cannot bind an axis bound to a formula, so the options are a fixed number or Excel's auto-scaling (which chose 0–120 on this data — wasteful and above the 100 ceiling a percentile can't exceed). The fixed floor is a deliberate readability trade-off, accepted by the owner. Current worst values sit 1.7 and 3.7 points above the floors.

---

## 8. Protection model (R15)

- **Locked = every formula**; unlocked = genuine inputs only.
- Mock Database editable cols: A, D, E, F, G, H, I, K, L, M, N, O, P, S, T, U, V, Y, Z, AA, AB, AF (rows 3–152).
- Calculations editable: **B38:B42** (targets + GP decay ratio). Lists editable: A2:D50. All output sheets fully locked.
- All 7 sheets `protect()`-ed **without password**; insert/delete rows & cols blocked; sort/filter/objects allowed.
- **Verified:** sheet protection does not break `FILTER` spills.

> **Maintenance note:** any scripted edit must read `protection.options`, `unprotect()`, write, then `protect(savedOptions)`. Re-applying `protect()` with no argument silently drops `allowSort` / `allowAutoFilter` / `allowEditObjects`.

---

## 9. Validation rules
Dates 2020–2035 · scores −100…200 (int) · percentiles 0–100 · attempts 0–150 (int) · Correct ≥0 & ≤ Attempts · Provider/Series/MockType/Analysis via `Lists` dropdowns (dynamic named ranges `lst_*` = `OFFSET`+`COUNTA`).

Series list (v1.2.0): SimCAT · Take Home · Previous CAT · DashCAT · Headstart CAT · Others.

---

## 10. Dashboard layout constraints

Row geometry is load-bearing. Verified heights:

| Rows | Purpose | Height |
|---|---|---|
| 1 | Title banner | 43.5 |
| 3 / 4 | KPI value / label | 37.5 / 12.8 |
| 5, 20, 24 | Spacers | 4.5 |
| 6 / 7 | Panel header / column header | 21.8 / 12.8 |
| 8–18 | Panel data | 16.5 |
| 19 | Insight strips | 19.5 |
| 25 | Mock Index header | 19.5 |

**Any new table in columns R:AB must occupy rows 8–19 or 26+.** Rows 3–7 have card geometry (including a 4.5pt spacer) that clips table rows. Columns S and T are shared with the Mock Index and are sized for mock names (84) and dates (70).

Merged regions: `A1`, `O1:P1`, `A19:H19`, `I19:P19`, `R8:V8`, `R25:AB25`, `A26`.

---

## 11. Named ranges

| Family | Pattern | Purpose |
|---|---|---|
| `lst_*` (4) | `OFFSET(Lists!X2,0,0,COUNTA(...),1)` | Dropdown sources |
| `cd_*` (10) | `OFFSET(Chart_Data!col2,0,0,COUNT(J),1)` | Packed ledger columns |
| `cn_*` (8) | same, over NA-mirror columns | Chart-safe mirrors |
| `cd_Idx` | `OFFSET(Chart_Data!$A$2,0,0,COUNT(J),1)` | **New in 1.2.0** — attempt index for GP weighting |

`cn_*` and most `cd_*` are vestigial — survivors of the rejected dynamic-name chart approach (§4). `cd_Ovrsc` and `cd_Idx` are live (GP average). Harmless, but candidates for cleanup.

---

## 12. Extension notes (for future maintainers)

- **Add a metric:** add a row on Calculations referencing a ledger column; surface on Dashboard by linking a panel cell. Anchor any windowed metric to `$B$13`, never a per-column `COUNT`.
- **Add a section:** extend Mock Database blocks, ledger cols, mirror cols, spill cols, and add chart series — follow the existing column-map pattern.
- **More than 150 mocks:** extend the `3:152` input ranges, the ledger's `MATCH` ranges, validation and conditional formatting. Spill/mirror already run to row 401; the progress block already runs to 400.
- **Extend GP weighting to other columns:** copy `B35` and swap the named range. Percentile columns work but are noisier (fewer observations).
- **Never** hand-edit `Chart_Data` — it's entirely derived.
- **Never** set `rowHeight`/`columnWidth` on a sub-range expecting it to be local. It isn't.
