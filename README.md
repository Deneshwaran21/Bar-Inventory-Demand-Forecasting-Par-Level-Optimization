# Bar Inventory Demand Forecasting & Par-Level Optimization

A forecasting and inventory-recommendation system for a multi-location hotel bar chain,
built to resolve simultaneous stockouts (lost sales) and overstocking (trapped capital)
across 6 bars and 16 product lines.

## Problem

The business symptom — short on fast movers, long on slow movers, at the same time —
pointed to an **allocation** problem rather than a volume problem. The goal was to
replace ad-hoc, per-bar reordering with a data-driven par level (target stock level) for
every bar-item combination.

## What this project demonstrates

- **Data extraction from an unstructured source** — the dataset arrived as a 165-page
  PDF export of a spreadsheet; built a parser to recover a clean, structured table from
  raw layout-preserving text.
- **Diagnose before modeling** — ran statistical tests (ledger-continuity checks,
  Kruskal-Wallis for seasonality, autocorrelation) to determine the true shape of the
  data *before* choosing a model, which ruled out a standard time-series approach in
  favor of intermittent-demand methods.
- **Bias correction** — identified that ~15% of records were right-censored by
  stockouts (recorded sales understated true demand) and corrected for it statistically,
  rather than training on biased history.
- **Model selection via rigorous backtesting** — benchmarked 9 forecasting methods
  (naive baselines, moving averages, Croston/SBA, gradient boosting) using rolling-origin
  validation, and selected the simplest model that matched top performance.
- **Business translation** — converted forecasts into an actionable decision (par
  levels, reorder points, safety stock) using an inventory policy framework (ABC
  segmentation, service-level targets).
- **Validation via simulation** — tested the recommended policy against real held-out
  data in a day-by-day simulation, rather than trusting forecast-error metrics alone.
- **Stakeholder-ready delivery** — packaged results for three audiences: a technical
  notebook, a business-facing written report, and a live, formula-driven Excel workbook
  with charts for non-technical managers.

## Repository contents

| File | Purpose |
|---|---|
| `bar_inventory_forecasting.ipynb` | Full analysis: data parsing, diagnostics, forecasting, par-level engine, simulation |
| `REPORT.md` | Business-facing write-up: problem, approach, results, limitations |
| `WRITEUP.md` | Structured answers to standard project-review questions (assumptions, model choice, production plan) |
| `par_levels.csv` | Output — recommended par level, reorder point, and ABC class per bar-item |
| `par_levels_insights.xlsx` | Interactive Excel workbook with live formulas and charts, for business users |
| `consumption_dataset_extracted.csv` | Cleaned dataset extracted from the source PDF |

## Approach summary

1. **Extract** — parsed the source PDF into a structured ledger of opening/purchase/
   consumed/closing stock per item per day.
2. **Diagnose** — verified the ledger was a complete movement record (not sampled
   readings) and confirmed demand is intermittent with no weekday/monthly seasonality.
3. **Correct** — imputed true demand on stockout-censored records using a lognormal
   tail-expectation model.
4. **Forecast** — backtested 9 models on a rolling-origin basis, scored on the actual
   decision horizon (lead time + review period), not daily accuracy.
5. **Decide** — built a periodic-review (R,S) inventory policy with safety stock sized
   correctly for intermittent (compound) demand, and ABC-based service-level targets.
6. **Validate** — simulated the policy against the real holdout quarter and compared it
   to what the bars actually did.

## Results (held-out quarter, vs. actual historical performance)

| Metric | Actual | Recommended policy |
|---|---|---|
| Fill rate | 97.7% | 99.0% |
| Stockout days | 52 | 24 |
| Average inventory held | 2,811 ml/item | 1,538 ml/item |
| Total cost (holding + lost sales) | $1,037 | $492 |

Every bar improved on both service **and** inventory simultaneously, confirming the
root cause was mis-allocation rather than under-buying.

## Tech stack

Python (pandas, numpy, scikit-learn, scipy), Jupyter, openpyxl, pdftotext.

## Limitations & next steps

See `REPORT.md` §8 and `WRITEUP.md` §4/optional section for a full discussion of
assumptions (e.g., assumed 2-day lead time), what would need validation with real
operational data, and what would need to change to run this at chain scale.
