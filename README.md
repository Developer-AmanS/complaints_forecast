# Complaints Volume Forecast - 90-Day Daily Prediction

---

> **If you read nothing else, read this.**

I built a system that looks at three years of daily complaint records and
predicts how many complaints the organisation will receive each day for the
next 90 days. The forecast comes with a range not just a single number -
so teams can plan for the realistic worst case, not just the average.

**What the data showed us:**

- Complaint volumes are growing at roughly **14% per year**.
- Volume is highest on **Monday–Wednesday** and lowest on **Friday**.
- There is a strong **seasonal pattern**: complaints peak in **December–February**
  (winter) and fall to their lowest in **June–July** (summer).
- These three patterns - trend, weekly rhythm, and annual cycle - account for
  most of the predictable structure in the data. The rest is day-to-day noise
  that no model can reliably predict.

**What I built:**
I tested four forecasting approaches (three simple baselines and one more
complex statistical model) against 90 days of real data the model had never
seen. The simpler rule-based model performed just as well as the more
sophisticated one, so I shipped the simpler version. A fancier model that
offers no accuracy improvement over a simple one only adds maintenance cost.

**The headline output:** `outputs/forecast_90d.csv` - one row per day from
2026-01-01 to 2026-03-31, with a middle estimate and an 80% confidence range.

---

## 1. Who this is for

This forecast is designed to support three different teams, each of whom
needs it at a different time horizon:

| Team                  | Horizon         | How they use it                                                                     |
| --------------------- | --------------- | ----------------------------------------------------------------------------------- |
| **Triage lead**       | Next 1–7 days   | Decide how many staff to have on the phones this week; balance the intra-week queue |
| **Rostering**         | Next 2–6 weeks  | Approve or deny leave; bring in contractors before a peak period                    |
| **Capacity planning** | Next 1–3 months | Make the business case for new hires; set budget assumptions                        |

**An important note on using the range, not just the middle number.**

The cost of getting the forecast wrong is not symmetrical. If we
_over-forecast_ - predict more complaints than actually arrive - the
consequence is some wasted capacity, which is recoverable. If we
_under-forecast_ - predict fewer complaints than arrive - the consequence is
a backlog. Backlog creates its own extra complaints (people chasing up
unresolved cases), making the problem worse. This feedback loop is why
**triage and rostering teams should always plan to the upper end of the
forecast range**, not the middle.

---

## 2. What the data told

The dataset covers daily complaint records from **1 January 2023 to
31 December 2025** - just over three years. Before building anything, I
examined the data to understand its structure. Here is what was found:

### 2.1 Complaints are growing year on year

| Year | Average daily complaints |
| ---- | ------------------------ |
| 2023 | ~67                      |
| 2024 | ~78                      |
| 2025 | ~96                      |

That is roughly **14% growth per year**. Whether this is driven by a growing
customer base, a change in product or policy, or increasing awareness of the
complaints process is outside the scope of this model - it forecasts _volume_,
not causes. But the trend is clear and must be built in.

### 2.2 There is a strong weekly rhythm

Complaints are not evenly spread across the week:

- **Highest:** Monday, Tuesday, Wednesday
- **Lowest:** Thursday, Friday, with Saturday and Sunday in between

The difference between the peak day (Monday, ~86 on average) and the trough
(Friday, ~72 on average) is around **20%**. This is large enough to matter
for daily staffing decisions - a flat "weekly average" forecast would
systematically under-staff Monday mornings and over-staff Friday afternoons.

### 2.3 There is a strong annual cycle

Complaints are highly seasonal:

- **Peak months:** December, January, February - roughly 30–40% above the
  summer average
- **Trough months:** June, July - the quietest period of the year

This matches common patterns for customer-facing services in the UK: people
have more time to pursue complaints over the winter, and there may be
seasonal product or billing cycles at play.

### 2.4 Some fields in the data look informative but aren't safe to use

The dataset contains several operational fields - staffing levels, backlog
days, media mentions, and a channel mix index. These show some correlation
with complaint volumes, but there are two problems with using them as
forecast inputs:

**Problem 1: We won't know their future values.**
A forecast for the next 90 days needs inputs that are knowable 90 days ahead.
We don't know what staffing levels or backlogs will be in March - those are
themselves outcomes of how the organisation responds to whatever volume
arrives. Using them would be circular.

**Problem 2: The direction of causality is likely wrong for some of them.**
The staffing level correlates with complaints because _high complaints cause
management to staff up_ - not the other way around. A predictor that is
itself caused by what you're trying to predict will make a model look
accurate in testing and then fail in production. I visualised this using
a cross-correlation analysis (§5 of the notebook), which confirmed staffing
consistently _lags_ complaints rather than leading them.

The only inputs safe to use are things knowable from the calendar alone:
day of week, time of year, and whether the day is a bank holiday.

### 2.5 One column in the data leaks the future

The column `centered_7d_mean` looks like a 7-day rolling average. What the
"centred" part means is that it uses three days _before_ and three days
_after_ each date to calculate the average. This means today's value of
`centered_7d_mean` contains tomorrow's, the day after's, and three days
ahead's complaint count.

This is called **data leakage**: a model trained on this feature would look
extremely accurate because it is, in effect, peeking at the future. In
production, there would be no future data to peek at, and performance would
collapse. We test for this explicitly (§6 of the notebook) and confirm the
column fails the test.

---

## 3. How to decide what to include

The rule I applied to every potential feature was simple:

> **Can we know this value on the day we run the forecast, for any day in the
> next 90 days?**

If the answer is no, it does not go in. This ruled out every operational
feature in the dataset. What remains is the calendar:

| Feature used                 | What it captures                         |
| ---------------------------- | ---------------------------------------- |
| Day of week                  | The Monday–Friday volume rhythm          |
| Time of year (Fourier terms) | The smooth winter peak / summer trough   |
| Bank holiday flag            | Days with distinctly different behaviour |

**Why "Fourier terms" for the annual cycle?**
Rather than having a separate adjustment for each of the 12 months, we use a
mathematical representation of a smooth wave that rises and falls once a year.
This uses far fewer parameters (6 numbers instead of 11), is less prone to
overfitting, and better captures the gradual ramp-up into the winter peak
rather than treating it as a step change on 1 December.

---

## 4. How I built and tested the forecast

### 4.1 The testing approach

I held back the **last 90 days** of the dataset - October to December 2025

- and pretended I didn't have them. I ran each model on the preceding data
  and compared its predictions against what actually happened. This is a fair
  test because it mirrors the real forecasting task: predict the next 90 days
  using only the past.

Then I repeated this test from **four different starting points**, not just
one, to check that performance was consistent rather than a result of one
lucky period.

### 4.2 What was measured

Came to three pass/fail criteria upfront, before running any model:

**Gate 1 - Accuracy:** The candidate model must be at least **5% more
accurate** than the best simple baseline. A complex model that only matches a
simple one is not worth the extra maintenance. _(Technical: MAE improvement
≥ 5% over best baseline.)_

**Gate 2 - Honest uncertainty:** The forecast range must actually contain
the true value the right proportion of the time. We publish an 80% range, so
it should contain the true value **70–90% of the time** in back-testing. A
model that claims to be 80% confident but is only right 50% of the time is
misleading. _(Technical: empirical coverage of 80% prediction intervals
within [0.70, 0.90].)_

**Gate 3 - No weekday bias:** The model should not be systematically wrong
on a particular day of the week. _(Technical: DoW-level mean error < 5% of
overall mean.)_

If any gate fails, the complex model does not ship. The framework
automatically falls back to the best simple baseline. There are no overrides.

### 4.3 What was tested

Ran four models:

| Model                          | Plain-English description                                                                                                                                                                                       |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Baseline 1: Constant mean**  | "It will be the same as the recent average every day." The laziest possible forecast.                                                                                                                           |
| **Baseline 2: Seasonal naïve** | "It will be similar to the same day last year, scaled for recent growth." No fitting required - just look up last year.                                                                                         |
| **Baseline 3: OLS calendar**   | A simple statistical formula with a growth trend, plus adjustments for each day of the week and each month. Transparent and fast.                                                                               |
| **Candidate: SARIMAX**         | A more sophisticated model that explicitly captures trend, weekly rhythm, and annual seasonality as separate mathematical components. Takes longer to fit and harder to explain, but inspectable when it fails. |

I chose SARIMAX as the candidate (rather than machine learning approaches
like gradient boosting or neural networks) for one reason: **when it fails,
you can see why**. Each component - trend, weekly, annual - can be plotted and
inspected independently. For a small team running this weekly, diagnosability
matters more than marginal accuracy gains.

---

## 5. What was shipped and why

### 5.1 The result

The candidate SARIMAX model **did not pass the gates**:

**Gate 1 result:** SARIMAX achieved a mean daily error of 23.9 complaints on
the 90-day test period. The best simple baseline (OLS calendar) achieved
the same - 23.9. SARIMAX needs to be at least 5% better to justify its
complexity, and it was not. Both models are off by roughly 24 complaints per
day on average, across a series where daily values range from about 60 to 170.

**Gate 2 result:** SARIMAX's confidence ranges only captured the true value
60% of the time, against a target of 70–90%. The model was too confident -
its uncertainty estimates were too narrow for a series this noisy. In
practice this means the ranges would understate the risk to planners.

**Decision:** The framework automatically shipped the **OLS calendar baseline**
with uncertainty ranges derived from bootstrapping (see glossary).

### 5.2 Why this is the right outcome, not a failure

It might feel disappointing that the sophisticated model did not win. It is
not. The result means:

- The signal in the data (trend + weekly + annual seasonality) is fully
  captured by a simple, transparent model. There is no hidden complexity that
  requires a more powerful approach.
- The team inherits a model that any analyst can read, re-run, and reason
  about - not a black box.
- The gate framework worked as designed. I wrote the rules before modelling
  and the rules made the call. No one had to argue about it.

The forecasts from the shipped model are in `outputs/forecast_90d.csv`.

### 5.3 What the forecast shows for the next 90 days

The forecast covers **1 January 2026 to 31 March 2026**. Key expectations:

- **January and February will be high-volume months**, consistent with the
  winter seasonal peak observed across all three years in the training data.
  Expected daily volume is in the range of 100–125 complaints per day at the
  mid-point estimate.
- **March begins a seasonal decline** as the winter peak fades, dropping
  toward the spring baseline.
- The **80% range** on any given day spans roughly ±40 complaints from the
  mid-point - reflecting genuine day-to-day unpredictability that no model can
  eliminate.

---

## 6. Reading the forecast output

The file `outputs/forecast_90d.csv` has four columns:

| Column     | What it means                                                                                                                           |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `date`     | The calendar date                                                                                                                       |
| `p50`      | The middle estimate - half the time the true value will be above this, half below. Use this for budget conversations.                   |
| `lower_80` | The lower end of the 80% range. Only 10% of days should fall below this.                                                                |
| `upper_80` | The upper end of the 80% range. Only 10% of days should fall above this. **Triage and rostering should staff to this figure, not p50.** |

**What the 80% range means in plain terms:** if you ran this forecast 100
times on different 90-day periods, roughly 80 of those 100 days would see the
true complaint count fall inside the stated range. The other 20 would fall
outside - this is not a failure, it is the honest acknowledgement that
forecasting is not certainty.

---

## 7. What could go wrong

Every forecast model has failure modes. Being explicit about them is more
useful than pretending they don't exist.

### 7.1 Something unusual happens - a spike with no precedent

**What it looks like:** A single day's actual complaints are far outside the
forecast range. Often followed by several more elevated days before returning
to normal.

**Why it happens:** The model learned from three years of "normal" operations.
A viral media story, a major system outage, or a sudden regulatory change
creates a spike that has no equivalent in the training data. The model has
no signal for it and cannot predict it.

**What to do:** Do not immediately retrain the model on the spike data.
Teaching the model that "sometimes complaints randomly triple" will
permanently widen its uncertainty ranges and reduce accuracy on normal weeks.
Instead, record the event in the run log (the `outputs/manifest.json` file),
wait for the anomaly to wash out of the recent data window naturally, and
investigate separately whether the event represents a one-off or the start of
a structural change.

### 7.2 The underlying pattern changes permanently

**What it looks like:** The model is consistently wrong in the same direction
for two or more weeks running - always too high or always too low. The error
is not random noise; it is a sustained drift.

**Why it happens:** A policy change, a new complaints channel, a product
change, or a business restructure shifts the baseline in a way that the
historical data cannot anticipate.

**What to do:** Refit the model using only data from after the change. Flag
the updated assumptions clearly in the next stakeholder update so everyone
is working from the same baseline.

### 7.3 The data pipeline breaks

**What it looks like:** The notebook fails with an error about missing or
unrecognised columns. Or the forecast suddenly jumps to an implausible value
for no apparent operational reason.

**Why it happens:** The data feed from upstream systems changed - a column
was renamed, a unit changed (e.g. from individual complaints to grouped
cases), or a new data source was connected without updating the schema.

**What to do:** The notebook contains a schema validation check (§3) that
catches this at the start of every run and blocks the forecast rather than
silently producing a wrong one. Investigate the upstream data source before
re-running.

### 7.4 A simple model starts beating the forecast

**What it looks like:** On the monthly model review, one of the three simple
baselines performs better than the current production forecast.

**Why it matters:** It means the operational landscape has changed enough
that historical seasonal patterns are no longer a reliable guide - the model
has become stale.

**What to do:** The gate framework handles this automatically - if the
candidate fails Gate 1 on refit, the best baseline ships instead. But
sustained underperformance over three or more consecutive weeks is a signal
to investigate whether the model needs to be rebuilt from scratch with the
new data regime.

### 7.5 Backlog and forecast drift up together

**What it looks like:** The forecast is rising, actuals are rising, and the
reported backlog days are also rising - all at the same time, for several
weeks.

**Why it matters:** This is likely not a forecasting problem - it may be
that the organisation is in a demand-capacity spiral: rising backlog
generates chaser complaints, which add to volume, which increases backlog
further.

**What to do:** Escalate to operational leadership. The forecast is probably
reporting reality correctly; the intervention needed is operational, not
analytical.

---

## 8. Ownership and maintenance

| Responsibility                                                  | Owner                 |
| --------------------------------------------------------------- | --------------------- |
| Running the weekly refit and checking the output                | Data science team     |
| Ensuring the input data pipeline delivers clean, on-schema data | Data engineering team |
| Acting on the forecast (staffing, rostering decisions)          | Operations team       |
| Investigating when the forecast diverges from actuals           | On-call rotation      |

**Refresh cadence:** The notebook should be re-run weekly. Monthly, the team
should review the model comparison table (`outputs/model_comparison.csv`) to
confirm the production model is still outperforming the baselines.

**Alert threshold:** If the average forecast error over any rolling 14-day
window exceeds 1.5 times the typical day-to-day variation, that is a signal
worth investigating before the next weekly run.

---

## 9. What I would improve next

These are not wishlist items - they are the specific gaps identified during
this build, in rough order of value:

**1. Better uncertainty ranges (conformal prediction).**
The current uncertainty ranges are derived by assuming forecast errors look
like a symmetric bell curve. On this data they do not - errors are more
spread out than that, especially around seasonal peaks. Conformal prediction
is a technique that derives ranges from the actual distribution of past
errors, with no shape assumptions. It would almost certainly fix the
calibration problem that caused Gate 2 to fail for SARIMAX.

**2. A "what-if" layer for unusual events.**
The current model has no way to account for planned events - a known media
campaign, a scheduled system maintenance window, a regulatory announcement.
A scenario layer would let operations ask "if we expect a 50% increase in
media mentions next week, how does that change the forecast?" and get a
quantified answer rather than a gut feel.

**3. Separate forecasts per complaint type or channel.**
If the data can be broken down by complaint category or incoming channel
(phone, email, web, etc.), different channels may have very different
seasonality and trend profiles. A single aggregate forecast misses this.

**4. Productionising the notebook.**
A notebook is the right format for exploration and documentation. Once the
forecast is being used to make live staffing decisions, it should be
converted into a scheduled script with automated alerting, a structured
output store, and a proper test suite. The notebook structure has been
deliberately designed to make that transition straightforward.

---

## 10. Technical reference

This section is for technical readers who want the full model specification.

### Model specification

| Parameter                 | Value               | Rationale                                                                    |
| ------------------------- | ------------------- | ---------------------------------------------------------------------------- |
| Model class               | SARIMAX             | Structural decomposition, inspectable components                             |
| AR order (p)              | 1                   | PACF shows single significant spike at lag 1                                 |
| Differencing (d)          | 1                   | Series has a clear linear trend; one difference removes it                   |
| MA order (q)              | 1                   | ACF shows single significant spike at lag 1                                  |
| Seasonal AR (P)           | 1                   | ACF shows spike at lag 7 - weekly seasonal component                         |
| Seasonal differencing (D) | 0                   | d=1 already handles trend; additional seasonal diff would over-difference    |
| Seasonal MA (Q)           | 1                   | Seasonal MA term absorbs residual lag-14 structure                           |
| Seasonal period (s)       | 7                   | Weekly cycle confirmed by DoW profile in EDA                                 |
| Exogenous regressors      | Fourier terms (k=3) | Annual cycle is smooth; 6 Fourier parameters preferred over 11 month dummies |

### Validation methodology

- **Hold-out window:** final 90 days of the dataset, matching the forecast horizon
- **Rolling-origin backtest:** 4 origins at 30-day intervals, each with a 90-day forward window
- **Metric:** MAE (mean absolute error) - preferred over RMSE because it is interpretable as average complaints-per-day error
- **Gate 1 threshold:** candidate MAE < 0.95 × best baseline MAE (5% margin)
- **Gate 2 threshold:** empirical coverage of 80% CI within [0.70, 0.90]

### Excluded features and reasons

| Field                | Reason for exclusion                                                                     |
| -------------------- | ---------------------------------------------------------------------------------------- |
| `centered_7d_mean`   | Fails causality test (§6): centred window uses future observations                       |
| `staffing_level_fte` | Cross-correlation peak at negative lag confirms reactivity; not future-known             |
| `backlog_days`       | Reactive to complaints volume; not future-known                                          |
| `media_mentions`     | Not knowable 90 days ahead; plausible leading indicator for a scenario layer             |
| `channel_mix_index`  | Near-zero contemporaneous correlation; no established causal mechanism; not future-known |

### Output files

| File                           | Contents                                                                     |
| ------------------------------ | ---------------------------------------------------------------------------- |
| `outputs/forecast_90d.csv`     | Date, p50, lower_80, upper_80 for each forecast day                          |
| `outputs/forecast_plot.png`    | Visual of last 365 days history + 90-day forecast with interval band         |
| `outputs/model_comparison.csv` | MAE, RMSE, MAPE for all four models on the validation period                 |
| `outputs/backtest.csv`         | MAE and interval coverage per origin in the rolling-origin backtest          |
| `outputs/manifest.json`        | Provenance record: data hash, config, gate results, model shipped, timestamp |

### Glossary

**MAE (Mean Absolute Error):** The average gap between the forecast and the
actual value, in complaints per day. An MAE of 24 means the forecast was, on
average, 24 complaints off per day. Easier to interpret than RMSE.

**SARIMAX:** Seasonal AutoRegressive Integrated Moving Average with
eXogenous regressors. A statistical model that explicitly represents trend,
short-range autocorrelation, and seasonal patterns.

**Fourier terms:** Sine and cosine waves used to represent smooth repeating
patterns. Here, they encode the annual seasonal cycle as a continuous wave
rather than a per-month step function.

**P50 / 80% interval:** P50 is the median forecast - equally likely to be
too high or too low. The 80% interval is a range within which the true value
is expected to fall 80% of the time.

**OLS (Ordinary Least Squares):** A standard statistical regression. Here,
it fits a linear trend plus day-of-week and month adjustments. The "calendar
baseline."

**Bootstrapping:** Estimating uncertainty ranges by repeatedly resampling
from historical forecast errors, rather than assuming a particular
mathematical shape for those errors. More robust when errors are not
normally distributed.

**Conformal prediction:** A technique for constructing prediction intervals
with guaranteed coverage properties, without distributional assumptions.

**Data leakage:** When a model feature contains information from the future,
making the model look more accurate in testing than it will be in production.

**Reactive feature:** A feature that is caused by the variable you are
trying to predict, rather than causing it. Using reactive features as inputs
creates a false impression of predictive power.

---

## 11. How to run

### Requirements

Python 3.9 or later. All dependencies are in `requirements.txt`.

### First-time setup

```bash
# Clone the repo and move into the directory
git clone <your-repo-url>
cd complaints-forecast

# Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate          # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Running the forecast

```bash
jupyter notebook complaints_forecast.ipynb
```

Run all cells in order from top to bottom. The notebook is fully
deterministic - the same inputs always produce the same outputs
(`RANDOM_SEED = 42`).

### Expected outputs

After a full run, `outputs/` will contain:

```
outputs/
├── forecast_90d.csv        ← the forecast; use this
├── forecast_plot.png       ← visual overview
├── model_comparison.csv    ← how each model did on the test period
├── backtest.csv            ← rolling-origin test results
└── manifest.json           ← full provenance record for this run
```

### Repo layout

```
.
├── complaints_forecast.ipynb   ← the notebook
├── data/
│   └── daily_records.csv       ← input data
├── outputs/                    ← generated by the notebook
├── requirements.txt
└── README.md
```

---

_Last updated: May 2026. For questions, contact the data science team._
