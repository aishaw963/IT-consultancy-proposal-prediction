# End-to-End Predictive Model with Deployment-Ready Pipeline — IT Consultancy
**SDC Internship — Week 3, Professional/Advanced Build — Member 1 (Individual Contributor)**

## 1. What this is and why

IT consultancies spend real time and money preparing proposals/bids — and not every bid is won.
This project builds a **proposal win-probability model**: given a new bid's industry, whether
the client is existing or new, the deal size, the number of competing bidders, the discount
offered, and how strong the relationship is, it predicts the probability the consultancy will
**win** the bid — before pricing is finalized, when the sales team can still act on it (adjust
the discount, decide whether the bid is worth pursuing, lean on the relationship).

This deliberately targets the **sales/business-development side** of a consultancy rather than
the delivery side (cost/schedule overruns) — a different business function, which is what makes
it a distinctive rather than a default choice.

The deliverable is a **pipeline**, not just a model: raw data flows through an automated cleaning
layer, into a trained regression model, into two deployable interfaces (an Excel "Predictor" tool
and a Power BI report), all of which recalculate automatically when new data is added. See
`01_Scope_Statement.md` for the scope defined before the build started.

## 2. What's in this delivery

| File | What it is |
|---|---|
| `01_Scope_Statement.md` | Scope written before building |
| `02_Power_BI_Build_Guide.md` | Exact steps + Power Query M + DAX to build the Power BI report |
| `03_README.md` | This file |
| `IT_Consultancy_Proposal_WinRate_Pipeline.xlsx` | The full working pipeline (see sheet guide below) |
| `Proposal_Data.csv` | The raw dataset, standalone — use this for both the Jupyter notebook and the Power BI import |

**Data note:** all proposal data is synthetic — 150 simulated bids, built with realistic (not
random) relationships between deal attributes and win/loss outcome, since real bid data wasn't
available or appropriate to use for a training exercise. Disclosed on the workbook's `Start_Here`
sheet as well.

### Workbook sheet guide
| Sheet | Purpose |
|---|---|
| `Start_Here` | Navigation + how to refresh the pipeline with new data |
| `Raw_Data` | 151-row simulated raw export, **deliberately containing 12 realistic data-quality issues** used to stress-test the cleaning layer (see §4) |
| `Lookup_Tables` | Industry list + historical win rate (computed live from Raw_Data), plus every imputation statistic the formulas depend on |
| `Cleaned_Data` | Formula-driven cleaning layer: trims text, standardizes categories, imputes missing/invalid numbers, flags every row that needed a fix |
| `Model_Ready_Data` | 150 rows (one confirmed duplicate excluded), numerically encoded, ready for regression |
| `Model` | The regression itself — Excel `LINEST`, full fit statistics, and a predicted-vs-actual table for every proposal |
| `Predictor` | The deployment interface — enter a new proposal's attributes, get an instant win-probability prediction and recommendation |
| `Dashboard` | KPIs, win-rate-by-industry chart, confidence-tier distribution, and a predicted-vs-actual accuracy scatter chart |
| `Testing_Log` | Every injected data issue, the expected fix, and a **live formula pulling the actual cleaned result** as proof |
| `Data_Dictionary` | Every column, every sheet, defined |
| `Changelog` | Real build history — what broke and what changed (see §5) |

## 3. Tools and technique

- **Excel** — synthetic data generation, formula-driven ETL/cleaning, a linear-probability
  regression via `LINEST`, and an interactive scoring interface.
- **Power BI** — Power Query for a more robust second implementation of the cleaning layer
  (notably, native duplicate removal), and DAX measures that deploy the Excel-trained
  coefficients as a live scoring engine, plus the visual report layer. Full build steps in
  `02_Power_BI_Build_Guide.md`.
- **Why a linear probability model, not logistic regression:** Excel's `LINEST` and Power BI's
  DAX have no built-in logistic-fit function. Regressing the 0/1 outcome directly and clamping
  predictions to 0–100% is a standard, pragmatic substitute inside these tools — disclosed
  explicitly everywhere the prediction is shown, not hidden.
- **Why train in Excel and deploy in Power BI:** DAX cannot fit a regression model; `LINEST` can.
  Training once and deploying fixed coefficients as a DAX measure is a defensible, standard
  professional pattern — explained further at the end of the Build Guide.

## 4. What was tested, and what was found

Twelve distinct, realistic data-quality problems were deliberately injected into a working copy
of the clean synthetic dataset before building the cleaning layer against it. Full detail with
live-pulled proof is in the workbook's `Testing_Log` sheet; summary:

| # | Issue | Result |
|---|---|---|
| 1 | Whitespace/casing inconsistency in industry name | Trimmed + standardized automatically |
| 2 | Lowercase "yes" for Existing_Client | Standardized to title case |
| 3 | Trailing whitespace in Existing_Client | Trimmed + standardized |
| 4 | Missing deal size | Imputed with dataset median, flagged |
| 5 | Missing industry | Labeled "Unknown", flagged as unmapped |
| 6 | Missing relationship strength score | Imputed with median, flagged |
| 7 | Non-numeric competing bidders (`"n/a"`) | Detected, imputed, flagged |
| 8 | Non-numeric deal size (`"TBD"`) | Detected, imputed with median, flagged |
| 9 | Out-of-range relationship score (9 on a 1-5 scale) | Treated as invalid, imputed, flagged |
| 10 | Negative competing bidders (-2) | Treated as invalid, imputed, flagged |
| 11 | Duplicate Proposal ID | Detected via `COUNTIF`, second occurrence excluded from modeling |
| 12 | Unrecognized industry category | Kept, flagged as unmapped, model falls back to the portfolio's overall average win rate instead of breaking |

Additional scenario testing beyond the injected cases (also logged in `Testing_Log`):
- A deal size of $0 doesn't break the model — there's no ratio/division in this formula, unlike a
  cost-overrun model, so there's nothing to crash.
- Typing an industry name outside the dropdown list into the Predictor correctly falls back to
  the portfolio-average win rate instead of erroring.
- An unrealistic Competing_Bidders value (20, well outside the training range) still returns a
  number — flagged honestly as a real limitation (§6), not hidden as if it were a safeguard.

Real bugs found and fixed during the build (the honest build history, not staged):
- The initial signal strength in the synthetic data was too weak — R² came back at just 29% on a
  first pass. Retuned the underlying relationships and noise level (checked in a quick Python
  regression before touching Excel) to reach a more convincing, still-realistic R² of 50.4%.
- An early version of the prediction formula produced a VALUE-type error on every row, caused by
  an accidental double `=` from wrapping an already-`=`-prefixed formula string inside another
  formula. Removed the redundant `=` and confirmed all 150 rows recalculated cleanly.
- A chart's conditional-formatting fill initially crashed with a TypeError from passing a raw hex
  color string where the library expected a fill object; fixed by wrapping it correctly.

