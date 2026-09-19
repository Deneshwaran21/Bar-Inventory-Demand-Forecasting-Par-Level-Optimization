# Bar Inventory: Demand Forecasting & Par-Level Recommendations


## 1. The problem, restated

The chain is short of its fast movers and long on its slow movers at the same time.
That combination is diagnostic: it is not a purchasing-volume problem, it is an
allocation problem. The same working capital, pointed at the right items in the
right bars, fixes both symptoms at once — and the held-out simulation below confirms
that, improving service *and* cutting stock rather than trading one for the other.

The data covers 6 bars × 16 brands = 96 bar-item combinations, 6,575 movement rows
over calendar 2023. An "item" throughout is a brand at a specific bar, because
Grey Goose at Smith's and Grey Goose at Taylor's are separate stocking decisions.

## 2. What the data actually is (and why it changed my approach)

I ran three diagnostics before choosing any model. Each one killed an approach that
would otherwise have been the obvious default.

(a) The ledger is complete, not sampled. Each item appears on only ~68 of 365 days,
which could mean either that readings are periodic snapshots of continuous trade, or
that rows exist only on days something moved. These imply completely different models,
so I tested it: `opening(t)` equals `closing(t−1)` for 98.0% of consecutive row
pairs (max drift 4.98 ml, i.e. spreadsheet rounding), and consumption is uncorrelated
with the gap since the previous row (r = +0.011) — a row covering 8 days shows no
more volume than one covering 2.

So absent days are genuine zero-movement days, and demand is intermittent: a draw
occurs on ~16% of calendar days, averaging 358 ml when it does. Fitting ARIMA or
Prophet to a "daily series" here would be modelling a series that is 84% structural
zeros.

(b) A sixth of the rows carry a stockout signal. 910 rows show zero consumption —
but 81% of those had under 100 ml on the shelf. They are not quiet days, they are
empty-shelf days. A further 264 draws ran the shelf to exactly zero, i.e. are
right-censored: the ledger recorded what was *sold*, not what was *wanted*. 74 of 96
items were affected at least once.

This matters more than its size suggests. Training on censored history teaches the
model to under-forecast precisely the items that keep running out, which sets their par
levels too low, which causes more stockouts. It is the feedback loop that keeps a chain
permanently short of its best sellers, and it is invisible unless you look for it.

(c) There is no seasonal or trend signal to forecast. Kruskal–Wallis across
weekdays gives p = 0.75, across months p = 0.14; lag-1 autocorrelation of
successive draw sizes is 0.16. Draw sizes are noisy (CV ≈ 0.6) around a level that
barely moves.

The strategic implication: forecasting effort has near-zero marginal return on this
data past a well-estimated demand *rate*. The leverage is in the inventory policy
layer and in the censoring correction — not in a more sophisticated forecaster. I
built the full model bake-off anyway, because that is a claim that has to be earned on
a backtest rather than asserted.

## 3. Demand reconstruction

Censored draws are replaced with `E[X | X > c]` under a per-item lognormal fitted on
uncensored draws only, iterated three times so the fit isn't itself dragged down by the
censored values (a light-touch Tobit correction).

Chain-wide this lifts estimated demand by +2.9% (1,969 L recorded → 2,026 L), but
on censored rows themselves the uplift is +91% — roughly double. The correction is
small in aggregate and large exactly where the decisions go wrong.

## 4. Forecasting: what was tested and what won

What is forecast. Par levels depend on demand over the protection interval
`H = lead time + review period = 9 days`, not on any single day. So the backtest grades
9-day total demand per item. Grading daily accuracy would reward a model for predicting
structural zeros — flattering and irrelevant to the decision.

Validation. Rolling-origin, 12 fortnightly cutoffs from July 2023, training only on
data at or before each cutoff. No shuffled splits.

| Model | MAE (ml) | WAPE |
|---|---|---|
| Gradient boosting (calendar + lag + intermittency features) | 340 | 0.68 |
| Shrunk item mean (selected) | 344 | 0.69 |
| Global mean rate | 346 | 0.69 |
| Item mean rate | 348 | 0.70 |
| Moving average, 91d | 356 | 0.71 |
| Syntetos–Boylan (SBA) | 361 | 0.72 |
| Croston | 368 | 0.74 |
| EWMA (α=0.05) | 374 | 0.75 |
| Moving average, 28d | 382 | 0.76 |

Reading this honestly. Every model lands within 13% of every other, and a
gradient-boosted tree with full calendar and lag features beats a *constant* by 2%.
That is not a failure of modelling — WAPE ≈ 0.68 is the irreducible noise floor
here. A 9-day window contains one or two draws, and nothing can predict which days they
land on.

What does help is pooling. Items differ far less than the noise implies, so
shrinking each item's mean toward the chain mean (λ ≈ 0.5, tuned on the backtest) beats
both the raw item mean and the global mean. Croston and SBA are the textbook-correct
family for intermittent demand and are unbiased on average, but their interval
estimator is noisy on one year of history, so they land mid-table.

Selected: the shrunk item mean. Within 1.5% of the boosted tree, with no training
step, explainable to a bar manager in one sentence ("your usual rate, nudged toward what
comparable outlets do"), and it handles cold start for free — a new bar or newly listed
brand simply starts at the chain rate and blends toward its own history. The bake-off
stays in the pipeline as a monthly re-check: on a chain with real seasonality I would
expect the tree to pull ahead, and the system should notice when it does.

## 5. From forecast to par level

A forecast is not a decision. The par level is the number the manager uses: top the
shelf back to this level at each weekly count.

Policy. Periodic-review order-up-to `(R, S)` — matching how bars really run, on a
standing weekly count and delivery rather than continuous monitoring.

| Parameter | Value | Basis |
|---|---|---|
| Review period `R` | 7 days | weekly stock-take per bar |
| Lead time `L` | 2 days | assumed — not present in the data |
| Protection interval `H` | 9 days | stock must last until the *next* delivery lands |
| Pack size | 500 ml beer, 750 ml wine/spirits | orders round up to whole bottles |
| Service level | 98 / 95 / 90% by ABC class | volume-weighted differentiation |

Safety stock under intermittency. The standard `z·σ·√H` assumes roughly normal
daily demand. Here demand is compound — a draw occurs with probability `p`, with size
mean `μ` and variance `σ²` — so the correct variance over `H` days is
`H·[p·σ² + p(1−p)·μ²]`. That second term is the intermittency penalty, and omitting it
understates required stock badly for lumpy items. Because even the compound-normal form
misses the right tail, the engine takes the safety factor from the empirical
distribution of historical 9-day windows, re-centred on the forecast rate, with the
closed form as fallback for thin history.

Output (`par_levels.csv`): one row per bar-item with forecast rate, ABC class,
service level, par in ml and in bottles, reorder point, and days of cover.

Two honest caveats on this section. First, the ABC split is nearly meaningless on
this dataset — item volumes span only 31–80 ml/day, so the classification is close to
a uniform service level. It stays in because real chains have heavily skewed volumes
and that is where most of the working-capital saving comes from, but I'm not claiming
credit for a segmentation this data didn't earn. Second, days of cover at par looks
high (~35 days) — that is a consequence of intermittency, not over-stocking: one draw
is ~6 days of mean demand, so an item that must not deny a guest has to hold several
draws' worth. The metric that matters is realised turns, below.

## 6. Simulation: what it would have been worth

Par levels are fitted on Jan–Sep 2023 and Q4 2023 is held out entirely. The
simulator steps day by day: receive deliveries, meet demand from stock, record the
shortfall when stock runs out, and on the weekly review day raise an order back up to
par, arriving after the lead time. The baseline is what the bars actually did over
the same period — their real closing balances and real stockouts.

| Metric (Q4 2023, 96 items) | Actual | With par levels | Change |
|---|---|---|---|
| Weighted fill rate | 97.7% | 99.0% | +1.3 pts |
| Stockout days | 52 | 24 | −54% |
| Items hitting ≥1 stockout | 38 | 18 | −53% |
| Average inventory per item | 2,811 ml | 1,538 ml | −45% |
| Inventory turns (annualised) | 7.4 | 13.6 | +83% |
| Total cost (holding + lost sales) | $1,037 | $492 | −53% |

Every bar improves on both axes simultaneously — the item-level scatter in the
notebook shows most items holding less stock *and* serving more demand. That is the
central finding, and it is what confirms the diagnosis in §1: the incumbent problem was
mis-allocation, so re-pointing the same capital fixes both sides.

Costing assumptions — $0.02/ml product cost, 25%/yr holding rate, $0.06/ml lost margin
— are placeholders stated so they can be swapped for finance's real figures. Relative
conclusions hold regardless; absolute dollars scale with them. The lost-sale penalty
deliberately excludes guest-experience damage, so it is a conservative floor.

Service level is a business decision, so the system prices it rather than hard-coding
it. Re-running the engine and simulation across targets:

| Target | Achieved fill | Avg inventory | Holding | Lost sales | Total |
|---|---|---|---|---|---|
| 90% | 98.1% | 1,173 ml | $143 | $569 | $712 |
| 95% | 99.0% | 1,335 ml | $163 | $299 | $463 |
| 98% | 99.7% | 1,664 ml | $203 | $87 | $291 |
| 99% | 99.8% | 1,799 ml | $220 | $68 | $288 |

Under the assumed cost ratio, total cost is still falling at 99% — the economics say
push service high, and the ABC step-down to 90% on C items is not paying for itself
here. That conclusion is entirely driven by the assumed lost-margin penalty, which is
exactly why it is an exposed input rather than a constant buried in the code.

## 7. How it runs in practice

Weekly cycle, per bar. Sunday night the ETL pulls the week's movement rows, rebuilds
the panel and re-imputes censored draws. Monday 06:00 the rates and par levels refresh
and each manager gets a count sheet: item, par in bottles, current on-hand, *order this
many*. The manager counts, the suggested order is pre-filled, and overrides require a
reason code (event, promotion, delisting) — those overrides are logged, because they are
the training signal for everything the model cannot see. Delivery lands Wednesday.
Mid-week, if an A item crosses its reorder point an alert fires for a top-up rather than
waiting for the next Monday.

Monitoring — the part that decides whether this survives contact with reality:

| Check | Trigger |
|---|---|
| Rolling 4-week fill rate per bar | below target 2 weeks running |
| Forecast bias per item | \|bias\| > 15% of rate over 8 weeks → rate is drifting |
| Censoring share | rising → pars too tight in the field |
| Days of cover | > 45 → dead stock; a delisting decision, not a reorder |
| Override rate | > 20% of lines → managers don't trust it; go find out why |
| Backtest re-run | monthly; promote the tree if it beats the shrunk mean by >5% MAE |

Orders are suggested, never placed automatically. The manager keeps the
outlet-level knowledge — a wedding block, a refurb, a local festival — that nothing in
this pipeline can see.

## 8. Assumptions and limitations

Assumptions made: 2-day deterministic lead time and weekly review (not in the data);
absent days are zero-demand days (tested in §2a); censored draws are lognormal
(reasonable given the observed right skew); 500/750 ml pack sizes; cost parameters as
stated; an item is a brand-at-a-bar; ABC thresholds at 70/90% of cumulative volume.

Limitations, ordered by how much they worry me:

1. Lead time is assumed. Lead-time *variance* usually drives more safety stock than
   demand variance does, and there are no PO dates here to estimate it. This is the
   largest unverified input in the system.
2. Costs are placeholders, so all dollar figures are directional.
3. One year, no observed seasonality. A single year cannot separate a real December
   lift from noise, and a hotel bar showing *no* calendar effect at all is odd enough
   that I'd suspect the data generation process. I would not carry "there is no
   seasonality" into production without a second year.
4. The baseline flatters the bars. Their observed fill rate uses censored
   consumption against imputed demand, so real lost demand was probably higher than the
   52 stockout days credited to them.
5. Substitution is ignored. A guest denied Grey Goose often accepts Absolut, so true
   lost revenue is lower than modelled and demand across the two is coupled.
6. No spoilage, breakage, theft, open-bottle pour, supplier MOQs or case-pack
   constraints beyond bottle rounding.

Next, in order of expected payoff: (1) join revenue and margin per item, so pars
optimise profit rather than volume and slow movers become a delisting decision;
(2) capture actual lead times from PO-to-receipt timestamps and make the protection
interval stochastic; (3) bring in PMS occupancy and event data — hotel bar demand should
track house occupancy, and that is the most likely source of genuinely forecastable
signal; (4) model within-category substitution; (5) run a 4-week A/B on two bars against
two matched controls before chain-wide rollout, with fill rate and inventory value as
the primary endpoints.
