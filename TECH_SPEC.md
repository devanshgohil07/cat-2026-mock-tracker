# CAT 2026 Mock Tracker — Technical Specification & Build History


**Audience:** an engineer, AI assistant, or maintainer who needs to understand this workbook end-to-end — philosophy, architecture, every formula, and the reasoning behind each design decision, including the dead-ends that were rejected and why.

---

## 0. Philosophy

The workbook exists to answer one question continuously: *"Am I on track for my **CAT** target, and where do I fix things?"* Every design choice serves three non-negotiables:

1. **Auditability** — a human can click any number and trace it to source. No value is computed off-sheet and pasted.
2. **Single source of truth** — raw data lives in exactly one place (`Mock Database`). Everything else is a pure function of it.
3. **Zero-maintenance growth** — logging a mock must update *everything* (stats, charts, tables) with no manual range edits.

A recurring tension throughout the build: **Excel-and-Sheets portability** vs. **Excel-only power**. Rule adopted: all *formulas* stay portable (no `**LET**`, no `@`, no spill-only operators in the engine); Excel-specific mechanics are confined to the chart layer, which doesn't port to Sheets anyway.

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
| R8 | *Latest* must reflect one mock, blanks stayed blank | User |
| R9 | Targets (99.9%ile overall; VARC 55 / DILR 40 / QA 45 scores) | User |
| R10 | 7-band conditional formatting | User |
| R11 | Genuine extra insights where scope exists | User |
| R12 | True dynamic chart x-axis | User (critical) |
| R13 | Point-to-mock traceability | User |
| R14 | Visual coherence across all added tables | User |
| R15 | Input validation + lock all non-input cells | User |

---

## 2. Architecture

```
Mock Database  (raw inputs, rows 3-**152**)
      │  (Attempt Order = join key)
      ▼
Chart_Data  (ledger + mirrors + spill sources)   ← internal plumbing
      │
      ├──► Calculations  (metric engine: B:I grid × metric rows, + targets/trends)
      │          │
      │          ├──► Dashboard  (cards, panels, insight strips)
      │          ├──► Section Analysis
      │          └──► Provider Analysis
      │
      └──► Charts (9)  + Mock Index (Dashboard R:AB)
```

### 2.1 The ledger (`Chart_Data`)
A **compacted, packed** copy of completed mocks, one row per mock, no interior gaps, indexed by Attempt Order.

- **Col A** — index 1…**400** (always populated; structural).
- **Cols B–K** — the ledger proper, pulled by `**INDEX**/**MATCH**` on Attempt Order:
  `=**IFERROR**(**INDEX**('Mock Database'!$D$3:$D$**152**, **MATCH**($A2,'Mock Database'!$C$3:$C$**152**,0)),"*)`
  Blank source cells are guarded to return `**` (not `0`) via
  `=IF(**ISNA**(**MATCH**(...)),*",IF(**ISBLANK**(**OFFSET**('Mock Database'!$N$2,**MATCH**(...),0)),"",**INDEX**(...)))`.
  Column map: B=Mock name(D), C=Date(I), D=**VARC** Sc(M), E=**VARC** %(N), F=**DILR** Sc(S), G=**DILR** %(T), H=QA Sc(Y), I=QA %(Z), J=Ovr Sc(AE), K=Ovr %(AF).
- **Cols O–V — NA-mirror.** Same values but empties → `NA()`:
  `=IF($D2="",NA(),$D2)`. Used by charts so a missing percentile becomes a **line gap**, not a zero.
- **Cols Z–AH — **FILTER** spills** (the dynamic engine):
  `=**FILTER**(<mirror col>2:**401**, $B$2:$B$**401**<>"*, *")` → exactly N rows, auto-resizing.
- **Cols AJ–AL — radar source** (section × Avg%ile, Best%ile).
- **Cols AM/AN — scatter X source:** `AM = IF($B2="",NA(),$A2)` (NA-guarded attempt index).

### 2.2 The engine (`Calculations`)
A grid of **metric rows × 8 metric columns**. Column→ledger map: B→J, C→K, D→D, E→E, F→F, G→G, H→H, I→I.

Every statistic reads a **packed, blank-safe** ledger column, so no `>0` filters are needed and native functions skip blanks correctly.

---

## 3. Formula reference (the engine)

Let `R` = the ledger column range (e.g. `Chart_Data!$J$2:$J$**401**`), `O` = its header cell, `n` = the count cell (row 13).

| Metric (row) | Formula | Notes |
|---|---|---|
| Latest (2) | `=IF($K$2="*,**,IF(INDEX(R,$K$2)=**,*",INDEX(R,$K$2)))` | Anchored to one mock via `K2` (below). Blank stays blank. |
| Average (3) | `=IFERROR(AVERAGE(R),"")` | Skips blanks natively. |
| Median (4) | `=IFERROR(MEDIAN(R),"*)` | |
| Highest (5) | `=IFERROR(MAX(R),**)` | |
| Lowest (6) | `=IFERROR(MIN(R),**)` | |
| Variance (7) | `=IFERROR(VAR(R),*")` | Was broken (blank) in v1 — see §5. |
| Std Dev (8) | `=IFERROR(STDEV(R),"")` | |
| Roll Avg 5 (9) | `=IFERROR(AVERAGE(OFFSET(O,MAX(1,n-4),0,MIN(5,n),1)),"")` | Window keyed to count, not raw rows. |
| Roll Avg 10 (10) | `=IFERROR(AVERAGE(OFFSET(O,MAX(1,n-9),0,MIN(10,n),1)),"")` | |
| Improvement (11) | `=IFERROR(Latest - INDEX(R,1),"")` | vs first mock. |
| Best-till-date (12) | `=IFERROR(MAX(R),"")` | |
| Count (13) | `=COUNT(R)` | Drives rolling windows. |
| Last-5-vs-Avg (14) | `=IFERROR(RollAvg5 - Average,"")` | |
| Last-10-vs-Avg (15) | `=IFERROR(RollAvg10 - Average,"")` | |
| Consistency (16) | `=IFERROR(100*(1-StdDev/Average),"")` | Higher = steadier. |
| **Trend (32)** | `=IFERROR(SLOPE(R, Chart_Data!$A$2:$A$401),"")` | Points gained per mock. |
| **Projected next (33)** | `=IFERROR(FORECAST($K$2+1, R, Chart_Data!$A$2:$A$401),"")` | Linear extrapolation. |
| **Coeff. of Variation (34)** | `=IFERROR(100*StdDev/Average,"")` | Volatility, scale-free. |

**Latest anchor** `K2 = =**IFERROR**(**COUNT**(Chart_Data!$J$2:$J$**401**),"")` — because the ledger is packed, the count equals the row index of the newest mock. All *Latest* cells **INDEX** into that one row (R8).

### 3.1 Progress block (rows 22–31)
- PlannedTotal `=**COUNTIFS**(D<>"*, H<>**)`; Completed `=**COUNTIF**(J,*Completed")`; Pending = diff.
- CompletionPct, AnalysisDone/Pending/Pct analogous.
- DaysToCAT `=**DATE**(**2026**,11,29)-**TODAY**()`.
- **PaceNeeded (31)** `=DaysToCAT/Pending` — days available per remaining mock; compare to actual avg gap (**B17**).

### 3.2 Targets & readiness (rows 36–41)
Inputs **B38**:**B41** (Overall %ile 99.9, **VARC** 55, **DILR** 40, QA 45). Per row: Recent = RollAvg5; Gap = Target − Recent; Status `=IF(Recent="*,*—*,IF(Recent>=Target,*✓ On target*,IF(Trend>0,*Closing +*&**TEXT**(Trend,*0.0*)&*/mock*,*Behind")))`; Mocks@trend `=**ROUND**(Gap/Trend,0)`.

### 3.3 Attempt efficiency (rows 43–50)
Uses previously-unused Attempts/Correct/Incorrect/Accuracy columns via `**AVERAGEIFS**(..., Status,*Completed*)`. **Marks/Attempt** = avg score ÷ avg attempts. Auto-insight string flags most/least efficient section and most negatives.

---

## 4. The dynamic-chart problem (R6/R12) — full history

This was the hardest requirement and went through several rejected approaches. Documented in full because it's non-obvious.

**Symptom:** charts froze at the mock count they had when the add-in bound them; new mocks didn't appear.

**Root cause:** the Office.js **API** resolves any range or defined-name you bind to a **static address** and writes *that* into the chart's **SERIES** formula. A self-expanding reference is not preserved.

| Approach tried | Result | Why rejected |
|---|---|---|
| Bind to full padded range (e.g. `B2:B401`) | ❌ | A **category** axis reserves one slot per cell — empty, `""`, or `#N/A` alike. 380 blank slots crushed the data into the left 5%. |
| Dynamic named range (`OFFSET`) in SERIES | ❌ | Office.js resolves the name to a static address at bind time; doesn't stay dynamic. Confirmed by redefining the name and seeing no chart change. |
| `FILTER` spill + spill-ref binding (`Z2#`) | ❌ (false positive) | Appeared to work when shrinking the spill, but an end-to-end add-a-mock test proved the SERIES stored the resolved address, not `Z2#`. |
| **XY-Scatter with value x-axis** | ✅ | A **value** axis auto-scales to actually-plotted points and **skips `#N/A` pairs entirely**. Binding to the full 400-row range grows forever with no rebinding. |

**Final implementation:** all 8 trend charts are `XYScatterLines`. X = `Chart_Data!AM` (NA-guarded attempt index); Y = the NA-mirror columns (O–V). Verified end-to-end: injecting a 21st mock extended every chart to 21 points with **zero** rebinding; removing it contracted them back to 20.

**Trade-off accepted (**R13**):** a value axis can't show text labels, so the x-axis shows **attempt number**. The **Mock Index** (Dashboard R:AB) restores point-to-mock traceability, itself built on `**FILTER**` spills so it grows too.

**Office.js gotchas discovered:**
- Excel silently reverts an in-place line→scatter `chartType` change; each chart must be **deleted and re-added**.
- The enum must be the string `*XYScatterLines*` — `Excel.ChartType.xyScatterLines` is `undefined`.
- Chart operations must be done **one chart per `context.sync()`**; batching multiple axis operations caused Excel crashes.

---

## 5. Bugs found & fixed (audit trail)

| Bug | Impact | Fix |
|---|---|---|
| Variance/StdDev used broken `SUMPRODUCT`+`@` | Returned blank → Consistency blank | Rewrote to `VAR`/`STDEV` over packed ledger |
| `*>0*` filter used as *not blank* everywhere | Silently deleted valid low/negative scores and blank-percentile mocks | Removed; rely on native blank-skipping over packed ledger |
| `MEDIAN(IF(@...))` | Non-portable implicit intersection | Packed ledger makes plain `MEDIAN` correct |
| Rolling-avg window keyed to wrong rows | Roll-5 averaged ~10 values | `OFFSET` keyed to count cell (row 13) |
| *Latest* via `MATCH(MAX(date))` | Wrong row on tied dates; independent per column | Single anchor `K2`; all Latest INDEX one row |
| Chart_Data was a **static hand-pasted snapshot** | Charts missing newest mock; never updated | Replaced with formula-driven packed ledger |
| CF: stacked rules, no `stopIfTrue` | Max-and-above-avg cell got **white font on pale fill** | Orthogonal bands, explicit priority, dark-on-pale pairs |
| Provider Analysis labels hardcoded | Drift risk | Linked to `Lists` |
| Correct could exceed Attempts | Negative *Incorrect*, >100% accuracy | Validation guard `OR(Att="",Correct<=Att)` |
| Blank ledger percentile → `INDEX` returned `0` | False zeros in charts/stats | `ISBLANK(OFFSET(...))` guard returns `""` |

---

## 6. Conditional formatting (R7/R10)

**7-band diverging system**, applied to score cols (M,S,Y,AE), percentile cols (N,T,Z,AF), and accuracy cols (R,X,AD,AJ), rows 3–**152**. Priority = insertion order (first added = highest); all `stopIfTrue`; every rule guarded `<cell><>""` so blanks stay uncolored.

| Band | Condition | Fill / Font |
|---|---|---|
| 1 | `= MAX` | `#1B5E20` / white bold |
| 2 | `= MIN` | `#B71C1C` / white bold |
| 3 | `≥ AVERAGE + STDEV` | `#43A047` / white |
| 4 | `≤ AVERAGE − STDEV` | `#EF5350` / white |
| 5 | `> MEDIAN` | `#C8E6C9` / `#1B5E20` |
| 6 | `< MEDIAN` | `#FFE0B2` / `#E65100` |
| 7 | catch-all (= median) | `#FFF9C4` / `#616161` |

All thresholds are live functions of the column, so bands re-shade as data changes.

---

## 7. Charts inventory

| Chart | Type | X | Y series | Axis notes |
|---|---|---|---|---|
| 1 Overall Score | XYScatterLines | AM | U | y auto |
| 2 Overall %ile | XYScatterLines | AM | V | y 75–100 |
| 3 Section Scores | XYScatterLines | AM | O,Q,S | y from 0 |
| 4 Section %iles | XYScatterLines | AM | P,R,T | y 20–100 |
| 5–7 VARC/DILR/QA | XYScatterLines | AM | score (primary) + %ile (secondary 0–100, dashed) | dual axis |
| 8 All sections | XYScatterLines | AM | O,Q,S + **U (Overall, thick)** | y auto |
| Radar | RadarMarkers | section | Avg%ile, Best%ile | radial 0–100 |

Palette: **VARC** `#**2E7D32**`, **DILR** `#**1565C0**`, QA `#**C62828**`, Overall `#**4527A0**`, Percentile `#**EF6C00**`.

---

## 8. Protection model (R15)

- **Locked = every formula**; unlocked = genuine inputs only.
- Mock Database editable cols: A, D, E, F, G, H, I, K, L, M, N, O, P, S, T, U, V, Y, Z, AA, AB, AF (rows 3–**152**).
- Calculations editable: **B38**:**B41**. Lists editable: A2:**D50**. All output sheets fully locked.
- All 7 sheets `protect()`-ed **without password**; insert/delete rows & cols blocked; sort/filter/objects allowed.
- **Verified:** sheet protection does **not** break `**FILTER**` spills — an add-a-mock test grew ledger, spills, index, and charts to 21 with zero spill errors, then restored to 20.

---

## 9. Validation rules
Dates **2020**–**2035** · scores −**100**…**200** (int) · percentiles 0–**100** · attempts 0–**150** (int) · Correct ≥0 & ≤ Attempts · Provider/Series/MockType/Analysis via `Lists` dropdowns (dynamic named ranges `lst_*` = `**OFFSET**`+`**COUNTA**`).

---

## 10. Portability & environment
- Engine formulas are Excel-**2021**/**365** **and** Google-Sheets safe.
- Auto-growth depends on `**FILTER**` (spill) — needs Excel **365**/**2021**+ or Sheets. Older Excel shows data but won't auto-expand.
- Charts are Excel objects; if opened in Sheets they'd be rebuilt, but the underlying data model is intact.

---

## 11. Extension notes (for future maintainers)
- **Add a metric:** add a row on Calculations referencing a ledger column; surface on Dashboard by linking a panel cell.
- **Add a section:** extend Mock Database blocks, ledger cols, mirror cols, spill cols, and add chart series — follow the existing column-map pattern.
- **More than **150** mocks:** extend the `3:**152**` input ranges and the ledger's `**MATCH**` ranges; spill/mirror already run to row **401**.
- **Never** hand-edit Chart_Data — it's entirely derived.
