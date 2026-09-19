# Write-up: Bar Inventory Forecasting & Par-Level System
---

## 1. What is the core business problem, and why does it matter?

The chain is stocking out of fast movers and overstocking slow movers at the same
time. That combination is diagnostic: it is not a purchasing-volume problem, it is an
allocation problem. The same working capital, pointed at the right items in the
right bars, should fix both symptoms — and the held-out simulation confirms it does,
improving service while reducing stock rather than trading one against the other.

Why it matters commercially:

- A stockout is an unrecoverable sale. A guest denied their drink either substitutes
  down or stops ordering. The margin is lost that night, and in a hotel the damage
  carries into the review score, which costs far more than the pour.
- Overstock is trapped cash plus risk. Slow-moving bottles tie up working capital,
  occupy limited back-bar space, and expose the business to breakage, theft and (for
  beer and wine) spoilage.
- The decision is currently a guess made 96 times a week. Six bars × 16 brands, each
  re-ordered by eye. There is no per-item rate, no service-level target, no safety stock
  logic — so errors in both directions are guaranteed.

The system replaces that guess with a par level per bar-item: top the shelf back to
this number at the weekly count.

## 2. What assumptions did I make, and why?

Tested, not assumed — I verified these rather than taking them on faith:

| Assumption | How it was tested | Result |
|---|---|---|
| The ledger is complete; absent days are zero-movement days | `opening(t)` vs `closing(t−1)`; correlation of consumption with the gap since the previous row | Holds — 98.0% exact match, r = +0.011 |
| No weekday or monthly seasonality | Kruskal–Wallis across weekdays and months | No effect — p = 0.75 and p = 0.14 |
| Zero-consumption rows are stockouts, not quiet days | Share of those rows with <100 ml on hand | 81% were empty shelves |

Genuinely assumed, with reasoning:

- Lead time = 2 days, deterministic. No PO or receipt timestamps exist in the data.
  Typical for a local beverage distributor. This is the system's largest unverified
  input, and lead-time *variance* usually drives more safety stock than demand variance
  does.
- Weekly review (R = 7 days). Matches how bars actually operate — a standing count
  and delivery, not continuous monitoring. Together with lead time this sets the 9-day
  protection interval that par levels must cover.
- Censored draws are lognormal. Needed to compute `E[X | X > c]` for the
  un-censoring step. Justified by the observed right skew of draw sizes.
- An "item" is a brand at a specific bar. Grey Goose at Smith's and at Taylor's are
  separate stocking decisions with separate demand.
- Pack sizes: 500 ml beer, 750 ml wine and spirits. Orders must round to whole
  bottles to be actionable.
- Costs: $0.02/ml product, 25%/yr holding, $0.06/ml lost margin. Placeholders,
  stated explicitly so finance can swap in real figures. Relative conclusions hold
  regardless; absolute dollars scale with them. The lost-sale penalty excludes
  guest-experience damage, so it is a conservative floor.

## 3. What model did I use, and why not the others?

Selected: a shrunk item-mean demand rate — each item's own mean daily rate blended
toward the chain-wide mean (λ ≈ 0.5, tuned on the backtest).

What the data forced. Because the ledger is complete, demand is intermittent: a
draw occurs on ~16% of calendar days, averaging 358 ml. A "daily series" here is 84%
structural zeros, which rules out the reflex choice of ARIMA or Prophet — they would be
smoothing a series whose defining feature is when it *isn't* zero.

The bake-off. Nine candidates, rolling-origin validation, 12 fortnightly cutoffs,
graded on 9-day total demand per item (the protection interval — grading daily accuracy
would reward a model for predicting structural zeros):

| Model | MAE (ml) | WAPE |
|---|---|---|
| Gradient boosting (calendar + lag + intermittency features) | 340 | 0.68 |
| Shrunk item mean — selected | 344 | 0.69 |
| Global mean rate | 346 | 0.69 |
| Item mean rate | 348 | 0.70 |
| Moving average 91d / SBA / Croston / EWMA / MA-28d | 356–382 | 0.71–0.76 |

Why not the others:

- ARIMA / Prophet / SARIMAX — no seasonality, no trend, and an 84%-zero series.
  Nothing for them to model.
- Gradient boosting — won by 1.3%, which is within noise, and it beats a *constant*
  by only 2%. That is the clearest evidence available that there is no temporal
  structure to find. It costs a training step, a feature pipeline and explainability,
  and it cold-starts badly for a new bar or a newly listed brand. It stays in the
  pipeline as a monthly re-check: if it ever beats the shrunk mean by >5% MAE, real
  signal has appeared and it gets promoted.
- Croston / SBA — the textbook-correct family for intermittent demand, and unbiased
  on average here, but their interval estimator is noisy on a single year of history, so
  they land mid-table. I would revisit them with two or three years of data.
- EWMA and short moving averages — they chase noise. With no trend to track,
  recency-weighting is pure variance.

Why shrinkage wins: items differ far less than the noise implies, so pooling across
the chain beats both the raw item mean and the global mean. It also gives cold start for
free — a new bar starts at the chain rate and blends toward its own history as weeks
accumulate.

The part that actually moves the numbers. WAPE ≈ 0.68 is the irreducible noise floor
here; no forecaster escapes it. The gains come from two other places:

1. Un-censoring demand. 264 draws ran the shelf to exactly zero and 741 more rows
   are empty-shelf days, so the ledger records what was *sold*, not what was *wanted*.
   Correcting this lifts chain demand +2.9%, but +91% on the censored rows themselves
   — exactly the fast movers whose par levels were being set from suppressed history.
   This is the feedback loop that keeps a chain permanently short of its best sellers.
2. Correct safety stock for intermittent demand. The standard `z·σ·√H` assumes
   near-normal daily demand. Here it is compound, so variance over the protection
   interval is `H·[p·σ² + p(1−p)·μ²]`. The second term is the intermittency penalty, and
   omitting it understates required stock badly for lumpy items. The engine goes further
   and takes the safety factor from the empirical distribution of historical 9-day
   windows, re-centred on the forecast, because even the compound-normal form misses
   the right tail.

## 4. How does the system perform, and what would I improve?

Par levels are fitted on Jan–Sep 2023; Q4 2023 is held out entirely. The simulator
steps day by day — receive deliveries, meet demand from stock, record shortfalls, order
back up to par on the weekly review day. The baseline is what the bars actually did
over the same period, not a strawman.

| Metric (Q4 2023, 96 items) | Actual | With par levels | Change |
|---|---|---|---|
| Weighted fill rate | 97.7% | 99.0% | +1.3 pts |
| Stockout days | 52 | 24 | −54% |
| Items hitting ≥1 stockout | 38 | 18 | −53% |
| Average inventory per item | 2,811 ml | 1,538 ml | −45% |
| Inventory turns (annualised) | 7.4 | 13.6 | +83% |
| Total cost (holding + lost sales) | $1,037 | $492 | −53% |

Every bar improves on both axes at once, and the item-level scatter in the notebook
shows most individual items holding less *and* serving more. That is what confirms the
original diagnosis: the problem was mis-allocation, so re-pointing the same capital
fixes both sides.

Service level is exposed as a business input rather than hard-coded. Under the assumed
cost ratio, total cost is still falling at a 99% target ($288) versus 90% ($712) — the
economics say push service high, and the ABC step-down to 90% on C items is not paying
for itself on this data.

What I would improve, in order of expected payoff:

1. Join revenue and margin per item. Then par levels optimise profit rather than
   volume, and persistent slow movers become a *delisting* decision rather than a
   stocking one — which is where the real overstock saving sits.
2. Capture actual lead times from PO-to-receipt timestamps and make the protection
   interval stochastic. Biggest reduction in model risk available.
3. Bring in PMS occupancy and event data. Hotel bar demand should track house
   occupancy; that is the most likely source of genuinely forecastable signal, and its
   complete absence in this dataset is itself suspicious.
4. Model within-category substitution — a guest denied Grey Goose often takes
   Absolut, so true lost revenue is lower than modelled and the two demands are coupled.
5. A/B test before rollout: 4 weeks, two bars against two matched controls, with
   fill rate and inventory value as primary endpoints.

Known limitations: the ABC split is near-meaningless on this data (item volumes span
only 31–80 ml/day) and is retained only because real chains have skewed volumes; one
year cannot separate a real December lift from noise; the baseline flatters the bars,
since their fill rate uses censored consumption against imputed demand; and no spoilage,
breakage, theft, open-bottle pour, supplier MOQ or case-pack constraints are modelled
beyond bottle rounding.

## 5. How would this work in a real hotel?

The weekly cycle, per bar:

1. Sunday night — ETL pulls the week's movement rows; the panel rebuilds; censored
   draws are re-imputed.
2. Monday 06:00 — rates and par levels refresh. Each manager gets a count sheet:
   item, par in bottles, current on-hand, *order this many*.
3. Monday count & order — the manager counts, the suggested order is pre-filled, and
   overrides require a reason code (event, promotion, delisting). Overrides are logged,
   because they are the training signal for everything the model cannot see.
4. Wednesday — delivery lands; receipts post back automatically.
5. Mid-week — if an A item crosses its reorder point, an alert fires for a top-up
   rather than waiting for the next Monday.

Orders are suggested, never placed automatically. The manager keeps accountability
for outlet-level knowledge — a wedding block, a refurb, a local festival — that nothing
in this pipeline can see. A system that removes their judgment gets quietly ignored; one
that saves them the arithmetic gets used.

Integration. Par levels are the interface. They export to whatever the property runs
— a POS/inventory module, a procurement system, or a printed count sheet on a clipboard
— which means the system delivers value before any deep integration work is funded.

Rollout. Two pilot bars for four weeks against matched controls, then chain-wide.
Start at a uniform 95% service target and let the sensitivity curve, priced with real
margin data, drive it from there.

---

## Optional: what breaks at scale, and what I'd track in production

What breaks:

- The per-item Python loop. Fine for 96 items; at 50 properties × 200 SKUs (10,000
  series) the nightly job needs vectorised group-bys or Spark. The maths is unchanged —
  it is an engineering fix, not a modelling one.
- Cold start becomes the common case, not the edge case. At scale, new properties,
  new listings and seasonal menu changes dominate. Shrinkage handles this; a per-item
  trained model would not, which is a second reason the simple estimator was selected.
- The empirical-quantile safety stock needs ~120 days of history. New items fall
  back to the compound-normal formula until they accumulate it.
- Data quality dominates. At six bars a miscounted stock-take is visible; at fifty
  it silently corrupts par levels. Ledger continuity (`opening(t) == closing(t−1)`)
  becomes a daily automated data-quality gate, not a one-off diagnostic.
- Heterogeneity across properties. A city-centre hotel and a resort should not share
  one shrinkage pool. Hierarchical shrinkage — item → property → property-segment →
  chain — is the natural extension.

What I'd track in production:

| Check | Trigger |
|---|---|
| Rolling 4-week fill rate per bar | below target 2 weeks running |
| Forecast bias per item | \|bias\| > 15% of rate over 8 weeks → the rate is drifting |
| Censoring share | rising → par levels too tight in the field |
| Days of cover | > 45 → dead stock; a delisting decision, not a reorder |
| Override rate | > 20% of lines → managers don't trust it; go find out why |
| Data-quality gate | ledger discontinuities, negative balances, impossible pours |
| Backtest re-run | monthly; promote the boosted tree if it beats the shrunk mean by >5% MAE |

The two that matter most are override rate and censoring share. The first tells
you whether humans trust the system; the second tells you whether it is quietly
under-serving. A model can look healthy on MAE while failing both.
