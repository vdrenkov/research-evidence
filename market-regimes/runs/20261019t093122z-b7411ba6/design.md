# Observable market states and cryptocurrency forecasting

This study evaluates fixed cryptocurrency direction forecasts across observable
market states, testing conditional forecast quality, information beyond state
membership and trading participation. `study.yaml` specifies the complete design.

## Source and evaluation calendar

Forecast inputs come from the completed Paper 3 `model-comparison` run
`20261017t071326z-a2727d90`, retained in
`../model-comparison/runs/20261017t071326z-a2727d90/`. The source revision and hashes
are recorded in `study.yaml`; the framework revision used for this evaluation is
recorded separately at execution. No forecasting model is refitted or selected.

The evaluation interval is **[1 September 2022, 31 August 2025)** at 00:00 UTC:
the last origin is **30 August 2025**, and leap day is included. Fold boundaries are:

| Fold | First origin | End, exclusive | Nominal origins per asset |
| --- | --- | --- | ---: |
| 1 | 2022-09-01 | 2023-03-01 | 181 |
| 2 | 2023-03-01 | 2023-09-01 | 184 |
| 3 | 2023-09-01 | 2024-03-01 | 182 |
| 4 | 2024-03-01 | 2024-09-01 | 184 |
| 5 | 2024-09-01 | 2025-03-01 | 181 |
| 6 | 2025-03-01 | 2025-08-31 | 183 |

There are 1,095 nominal origins per asset. The repaired source hour on 24 March 2023
invalidates that day's target and the 29 subsequent input-dependent origins,
25 March–22 April. Thus 1,095 - 29 = 1,066 decision-eligible origins and
1,066 - 1 = 1,065 scored origins. The terminal date is a declared source boundary,
not another exclusion. A 31 August origin's delayed holding interval
would require a 1 September 01:00 quote outside the saved hourly stream.

Economic exposure uses **[1 September 2022 01:00, 31 August 2025 01:00)**:
26,280 hourly marks. Daily risk uses 1,096 marks including the terminal endpoint,
giving 1,095 daily returns. Each run exports the calendar reconciliation.

BTC/USDT, ETH/USDT and SOL/USDT share the source's nine model profiles, six rolling
24-month training windows, daily origins, next-day labels and three-seed probability
means. Assets, folds and seeds are not independent replications. Earlier results
and reviews informed this design: a fresh execution remains retrospective, not
independent confirmation or preregistration.

## Primary analysis

At midnight origin `t`, the available daily close is the previous 23:00 hourly
close. Bull requires a 90-day return of at least 10% and a positive five-day
change in the 30-day moving average; bear uses at most -10% and a negative change.
Other known cases are sideways. Volatility is the sample standard deviation of
30 daily log returns, annualized by `sqrt(365)`. Low/normal/high cutoffs are
training-only terciles, frozen by asset and fold. Calibration uses eligible
training origins, with at least 250 valid training state observations. Missing or
repaired daily closes invalidate dependent windows; no prices are filled.

For each model, excess Brier loss is model loss minus the training-up-rate reference
loss. Compute bull-minus-bear and high-minus-low conditional means within each
asset, then average the three asset contrasts equally: **18 primary contrasts**.
A negative contrast indicates lower relative loss in the first state, not positive
absolute skill. Each side requires at least 60 scored dates, three observed
episodes and three folds in every asset. Unscored gaps do not create episodes.
Ineligible comparisons remain in the declared family and are reported without intervals.

Use 100,000 synchronized circular calendar-block draws within folds: 14 days
preferred, 7 and 28 days sensitivity. Sample complete calendars before applying
score/state masks; use identical draws across assets and models. At least 95% of
draws must be jointly defined for eligible comparisons. Report ordinary 95% and
Bonferroni intervals over the 18 comparisons, valid counts and tail counts.
Inference is conditional on fixed forecasts and approximate dependence assumptions;
it does not include refitting uncertainty or establish equivalence.

## Review-motivated secondary analyses

**State-definition sensitivity.** Evaluate four one-factor alternatives: trend thresholds
of ±15%; a 60-day trend-return window; a 45-day volatility window; and volatility
quartile cutoffs. All other settings remain at their original values. Recompute
causal states, training cutoffs, support and state scores for each alternative.
Only the changed dimension contributes inferential contrasts: nine models for each
alternative, **36 comparisons in one secondary Bonferroni family**, including
ineligible cells. Apply the same support rules and all three block lengths, with
14 days preferred and at least 95% jointly valid draws within each alternative.
All alternatives are reported; trading gates remain unchanged.

**Training-only state probability references.** Under the original state definitions,
estimate separate trend and volatility references for each asset/fold. Use exactly
the source's common, eligible, final-refit training endpoints: label end must be
strictly before the first test origin. Apply that fold's training-derived state
cutoffs to causal training features. For a known state with at least 30 eligible
training labels, use `q = (positive labels + 1) / (training labels + 2)` (Laplace
smoothing, alpha = 1). For a sparse or unknown state, use the source's unconditional
training-up-rate. Estimates remain fixed throughout the fold; smoothing, support
thresholds and fallback rules are not selected on test outcomes. Retain state counts,
positive counts, fallback flags, training boundaries and the resulting probabilities.

For each model and each of the two state references, compute the mean paired Brier
loss difference over all common scored origins within each asset, then give assets
equal weight: **18 comparisons in a separate secondary Bonferroni family**. Include
fallback dates. Negative differences favour the model over that particular state
reference. Use the same 100,000 draws and 7/14/28-day blocks, with 14 days preferred.
Require at least 60 scored dates per asset and 95% jointly valid draws. Report
within-state absolute/reference scores descriptively, without additional intervals.
The two references are separate one-dimensional baselines, not a joint-state model.

The primary, sensitivity and state-reference families are distinct. Each controls
its own declared family at 5%; no 5% error guarantee is claimed over all three
families or choices of block length. The additional analyses remain review-motivated
secondary evidence.

## Secondary economics

Use the fixed 0.55 long/cash threshold with one-hour delay, holding interval
`[t + 1h, t + 25h)`, terminal liquidation and positions carried across folds.
Commission/slippage costs in basis points per side are 0/0, 6/5 (preferred), 12/10.
Repaired prices cannot supply fills or required daily marks; other flagged
intraday marks retain the source's accounting-only carry. The 100% current-bar
volume ceiling is ex post and must be checked for binding requests.

Gates use the primary state definitions: bull-only and avoid-high-volatility.
Each has an ungated cash-sleeve control whose active initial fraction is the permitted-state share
in the first fold's eligible training period. Remaining initial capital stays in
zero-interest cash, without transfers or fold resets. This fraction does not match
realized test exposure. Include ungated policies and passive buy-and-hold/cash
controls: `(9 × 5 + 3) × 3 assets × 3 costs = 432` descriptive replays. Report equity,
ROI, daily Sharpe (`sqrt(365)`), drawdown, fees, turnover and exposure together.
Do not infer timing skill, statistical significance or deployability from these
comparisons. The forecast and execution targets differ.

## Reproduction and outputs

The framework launcher is `scripts/run_market_regime_study.ps1`: Smoke checks source
preparation and synthetic comparisons; Full executes the complete evaluation using
the same committed code, design and environment. Neither uses earlier Paper 4 outputs.
Both repositories must remain clean and unchanged between Smoke and Full and during
any resumable execution. Commands and resume instructions are in the framework's
`docs/research.md`.

Each fresh `results/market-regimes/<run-id>/` directory retains source records,
state/reference estimates, code/configuration/environment identities, checkpointed
intervals and policy traces. Exports are `main-dataset.xlsx` and `tables.xlsx`.
