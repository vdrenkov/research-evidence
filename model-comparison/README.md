# Sequence models and tabular baselines for cryptocurrency forecasting

This document specifies the research question, methods and interpretation limits
for the `model-comparison` study. Exact model settings and execution controls are
defined in `study.yaml`.

## Question and contribution

Do sequence models improve next-day direction probabilities when tabular models
receive the same historical information? Does any improvement survive comparison
with a feed-forward neural network and a common trading policy?

The study separates three comparisons: adding historical observations to XGBoost,
changing the architecture while retaining those observations, and converting
probabilities into trades. A larger collection of models is not the contribution.
Results will describe the specified profiles, assets and period, rather than
establishing the superiority of an entire model family.

The historical data and earlier experimental results have already been examined.
This is a retrospective benchmark. Recording its design before execution does not
make the historical data unseen or constitute independent confirmation. The study
generates its own forecasts; previous forecasts are not reused as its results.

## Common forecasting task

| Item | Specification |
| --- | --- |
| Source | Existing Binance spot hourly OHLCV partitions, including repair flags. No new download or filling. |
| Assets | BTC/USDT, ETH/USDT and SOL/USDT, fitted separately. |
| Source interval | `[2020-09-01, 2025-09-01)`, UTC. |
| Forecast origin and target | Daily at 00:00 UTC, after the preceding day closes; predict whether the following 24-hour close-to-close return is positive. Zero return is class 0. |
| Outer test interval | `[2022-09-01, 2025-08-31)`, with six calendar folds of up to six months. The final fold is deliberately one day shorter to leave terminal execution data. |
| Training endpoints | Previous 24 calendar months. A training label must end strictly before the first test origin. |
| Historical input | 28 completed daily feature vectors, oldest first. Each vector contains five OHLCV-derived quantities defined below. |
| Common support | All models use the same eligible training endpoints, validation endpoints and scoring dates. The current-vector control also follows the 28-vector eligibility rule. |
| Randomness | Seeds 17, 42 and 73; primary forecasts average their probabilities. Seeds are not independent market replications. |

For a completed daily candle, the five features are the log close-to-previous-close
return, log high/low range, log close/open return, close location within the daily
range, and change in log(1 + daily base-asset volume). Close location is 0.5 when
high equals low. Return and volume change also require the preceding daily candle,
so a complete sequence depends on 29 completed source days. No indicators with
additional hidden lookbacks, external data or cross-asset inputs are included.
The training window bounds example endpoints; their sequences may use earlier
history where the stored source permits it. Initial warm-up exclusions remain
visible. Original provider retrieval dates are not reconstructed; the new run
records the exact stored inputs and the time they were frozen for execution.

Missing or synthetic hours invalidate affected daily candles and any dependent
feature sequence or target. Unknown repair flags fail preparation. There is no
new interpolation. Decision eligibility uses
past information only; scoring additionally requires an observed target. Retain
per-asset exclusions and the common masks. An unavailable target must not change a
decision that would have been made from complete past information.

The current store contains 20 flagged hours per asset, including one test-period
hour at 2023-03-24 13:00 UTC. For trading accounting, retain existing carried marks
with their flags; they are not observed prices and may not supply fills. Stop if
an intended fill or required daily valuation uses a missing/synthetic price.
Preparation conservatively checks every possible daily execution open at 01:00
UTC and each required valuation candle, before probabilities are known.
Other flagged intraday marks may carry the last observed close for bookkeeping;
do not invent a price path inside the gap. Flag the resulting intraday drawdown
limitation and retain gap diagnostics. This check must not use future missingness
to alter the forecast or its intended position.

## Models and training controls

| Model | Input | Purpose |
| --- | --- | --- |
| Logistic regression | Flattened 28 x 5 history | Linear historical baseline. |
| Random Forest | Same flattened history | Additional nonlinear tabular baseline. |
| XGBoost, current vector | Most recent five values only | Restricted-history control. |
| XGBoost, full history | Flattened 28 x 5 history | Main tabular comparator. |
| Multilayer perceptron | Same flattened history | Neural comparator without recurrent, convolutional or attention structure. |
| LSTM and GRU | Same 28 x 5 history | Recurrent profiles. |
| Temporal convolutional network | Same history | Causal convolutional profile covering the full window. |
| Transformer encoder | Same history with positional encoding | Attention profile. |

Exact fixed settings are in `study.yaml`. The information-matched models receive
identical values and ordering; flattening changes their representation, not the
available history. XGBoost's current-vector arm is the intentional exception.
Different dimensionality and inductive biases remain part of that comparison.

There is one architecture/profile per model, no architecture search, class
weighting, test-based tuning or probability recalibration. All models use a
training-only standardization of the five channels, shared across lag positions.
Fit its moments to unique daily vectors used by the training sequences, rather
than counting repeated overlapping vectors as additional observations. Constant
channels use scale 1. Keep all models' final training endpoints identical.

The last 90 calendar days of each outer training window are an internal validation
segment. In the selection stage, fit on earlier endpoints whose labels end
strictly before validation starts. Fit that stage's scaler on those inputs only.
Use unweighted validation log loss to select XGBoost's number of trees and the
neural models' number of epochs. Ties select the earlier iteration. Validation
labels must end strictly before the outer test origin.

Then discard selection-stage weights and refit from scratch on all eligible outer
training endpoints, including the validation segment, for the selected duration.
Refit the scaler on those final training inputs. Logistic regression and Random
Forest have fixed settings and need only this final fit. This avoids giving neural
models fewer final training examples. No outer test labels enter either stage.

There are **486 final model fits**: 9 models x 3 assets x 6 folds x 3 seeds.
There are additionally **378 duration-selection fits** for the two XGBoost arms
and five neural models, giving **864 estimator-training stages** in total.
Probability references at 0.5 and the full-training historical up-rate require no
estimator fits. Record actual counts, selected durations, parameter counts,
convergence, seed variation, timings, device and final sample counts. A training
failure stops the run; models or seeds must not disappear from the comparison.

CPU execution with one compute thread is the reference environment.
Request deterministic algorithms and record the actual hardware and libraries.
Runtime describes this environment; the design does not equalize parameter
counts, floating-point operations or optimization difficulty. Reproduction on
different hardware may need numerical tolerances; retained predictions preserve
the actual reported execution.

## Forecast evaluation

The primary outcome is mean Brier loss, first averaged equally across the three
assets at each common scoring date and then across dates. Do not treat assets,
seeds, folds or overlapping training windows as independent replications.

The nine planned contrasts are:

- Each of LSTM, GRU, TCN and Transformer minus full-history XGBoost: four contrasts.
- Each of those four sequence models minus the full-history MLP: four contrasts.
- Full-history XGBoost minus current-vector XGBoost: one context contrast.

Negative differences favor the first model. Report every contrast, ordinary 95%
percentile intervals and Bonferroni family intervals covering all nine. Use 50,000 synchronized
circular calendar-block bootstrap draws within outer folds, with 14-day primary
blocks and 7/28-day sensitivities. Share sampled dates across assets and models;
do not fill gaps by treating neighboring eligible rows as consecutive calendar
days. Apply one shared common-score mask after sampling calendar dates, recompute
the pooled valid-date mean in every draw, and reject a draw with no valid scores.
Use the same bootstrap draws for all contrasts at a given block length.

These are conditional intervals for retained fitted forecasts and a fixed asset
panel. They do not include retraining uncertainty, random asset sampling or all
dependence induced by overlapping training histories. Report fold and asset
estimates, individual-seed dispersion and bootstrap tail resolution. A nonsignificant
contrast is not equivalence. A sequence-specific advantage requires evidence
against both its XGBoost and MLP comparators, not just the restricted-history arm.

Secondary descriptive outcomes are log loss, ROC AUC, accuracy at 0.5, ten-bin
reliability summaries, skill against both probability references and computational
cost. Clip probabilities only when evaluating log loss, at 1e-7 and 1 - 1e-7.
Report unavailable AUC for single-class subsets. Do not select a winner from a
secondary metric or favorable asset, seed or sensitivity cell.

## Trading check

Trading is secondary and separate from forecast scoring. Apply identical daily
long/cash policies to the nine seed-averaged models and two probability references.
The primary threshold is 0.55, with 0.50/0.60 sensitivities. Retain buy-and-hold as
a separate benchmark. Each asset starts with 10,000 USDT; positions carry across
folds and terminal liquidation is charged.

Use the corrected hourly engine: a forecast at origin t may first fill at the
open at t + 1 hour. The intended holding interval is `[t + 1h, t + 25h)`.
The forecast label instead measures the close-to-close return over `[t, t + 24h)`;
neither is a substitute for the other. Missing decision histories request cash.
Keep actual source-decision and availability times and verify fills against them.

Per-fill commission/slippage scenarios are 0/0, 6/5 and 12/10 basis points; 6/5
is primary. Disable the engine's current-bar volume cap and volume-sensitive
slippage (`max_participation: 1.0`, `volume_sensitivity: 0.0`). These are assumed
small-capital executions at stored hourly opens, not a market-capacity model.
Fractional quantities are allowed; historical order books, lot filters and fee
tiers are not reconstructed.
Current-hour realized volume may be reported as an ex post diagnostic but must
not control fills or supply information at the open.

The complete grid has **306 policy replays**: 11 probability policies x 3 assets
x 3 thresholds x 3 cost scenarios, plus 3 buy-and-hold assets x 3 costs.
Report net return, daily Sharpe at common 24-hour valuation marks, drawdown,
exposure, turnover, fees and fill counts. The zero-risk-free Sharpe summary uses
sample standard deviation and sqrt(365) scaling; it is not a population guarantee.
Always show exposure beside trading comparisons. Lower drawdown with less time
invested is not evidence of better forecasts. Do not retune thresholds after results.

## Evidence and execution

This repository holds the study design and configuration. Implementation, tests
and the `run_model_comparison_study.ps1` launcher are in the
[crypto-backtest-framework](https://github.com/vdrenkov/crypto-backtest-framework).
Both repositories must be committed and clean before execution. The launcher
requires a software revision descended from the unchanged `v3.0.0` checkpoint;
the actual execution commit is recorded separately for every run.

Every run freezes this design/configuration, actual code/evidence revisions,
dependencies, hardware, seeds, normalized input snapshots with repair flags,
source-file identities and the eligibility/fold calendar before fitting. Each
execution has a fresh `results/model-comparison/<run-id>/` directory. PostgreSQL
and additional data downloads are unnecessary. Smoke and failed runs retain their
own identities; they are not relabeled as full evidence.

A completed run must retain actual fit and replay counts, exclusions, original
predictions, training records, timing, fills/equity, consolidated analyses and
completion/output manifests. A compact verification package in this repository
should contain identity/completion/output records, exact inputs, the calendar,
predictions and fit metadata. Larger artifacts remain in the framework's results
store, identified by the run records. Data sharing remains subject to provider
conditions.

### Completed execution

Run `20261017t071326z-a2727d90` completed 486 final fits, 378 duration-selection
fits and 306 policy replays. Its compact verification package is in
`runs/20261017t071326z-a2727d90/`: the archived design/configuration, run records,
exact inputs, eligibility calendar, original seed and ensemble/reference
predictions, and fit metadata.

The output manifest covers the full original run. Workbooks, figures, fitted
states and individual forecast/trading checkpoints remain in the framework's
`results/model-comparison/20261017t071326z-a2727d90/` directory. Their manifest
entries identify the separately retained files; they are not duplicated here.

### Commands

After both repositories are committed and clean, run from the framework checkout:

```powershell
./scripts/run_model_comparison_study.ps1 -StudyRoot '<absolute-path>/model-comparison' -Mode Smoke
```

The command freezes actual inputs and environment before fitting. It fits nine
profiles for the first asset/fold/seed: nine final fits and seven selection fits.
Prediction-path checks use five validation inputs already included in the final
refit and are deliberately unscored. The execution check uses synthetic prices.
No outer-test accuracy or profitability is computed. It prints a fresh run ID and
writes a separate smoke receipt and a rough fit-time scaling estimate, excluding
trading, bootstrap, I/O and differences in sample size or selected durations.

`-Mode Prepare` verifies and freezes without fitting. With a draft configuration,
it prepares a smoke run. Resume that preparation using `-Mode Smoke -RunId '<id>'
-Resume` on one command line. For interruptions, repeat the original mode with
`-RunId '<id>' -Resume`; completed model/seed jobs are verified and reused.

After smoke completion and inspection, change only `status` to `ready` and
`execution_ready` to `true`, then commit the configuration. Keep the study README,
scientific settings, code, dependencies, hardware and inputs identical to the
successful smoke check, or repeat it. The full command is:

```powershell
./scripts/run_model_comparison_study.ps1 -StudyRoot '<absolute-path>/model-comparison' -Mode Full -SmokeRunId '<successful-smoke-id>'
```

Resume Full with its own `-RunId '<full-id>' -Resume`; the original smoke linkage
is retained automatically. `-Mode Analyze -RunId '<full-id>'` requires every
forecast checkpoint and cannot fit missing models. For an already completed run,
these commands verify artifacts without rewriting them. A preparation failure
before an identity was fully archived needs a fresh ID; retain the failed folder.
Detected identity drift during execution invalidates that unfinished run, even if
the original checkout is restored: inspect it and preserve the discrepancy before
starting a fresh run. Do not reuse potentially mixed-source checkpoints.

During any active or resumable run, leave both repositories and the environment
unchanged. No code, inputs, predictions or hashes from previous runs are rewritten.

## Related work and limits on novelty

Comparing shallow and deep cryptocurrency predictors is established research.
The following studies help define the scope of this comparison:

| Study | Relevant overlap and distinction |
| --- | --- |
| [Murray et al. (2023)](https://doi.org/10.3390/forecast5010010), Sections 2.1-2.4 | Common 30-day lagged-price inputs and an incremental model comparison. Equal-history inputs alone are already established. Our task uses direction probabilities, an explicit context control and a feed-forward neural comparator. |
| [Omole and Enke (2024)](https://doi.org/10.1186/s40854-024-00643-1), methodology | Direction forecasting and trading are already jointly studied. Their on-chain feature-selection/DL comparison differs from our fixed OHLCV information comparison. |
| [Kaygın et al. (2026)](https://doi.org/10.3390/math14060989), pp. 10-12 | SVM, LSTM, GRU and hybrids predict price levels using a 70/30 split and seed 42. Reported recurrent input has one timestep. Our 28-step history comparison, proper probability scoring and rolling multi-seed design answer a narrower question. |
| [Vasileva et al. (2026)](https://doi.org/10.3390/appliedmath6060100), Sections 2.3 and 6.2 | Six linear/tree regressors forecast next-day returns with walk-forward/nested validation and examine statistical/economic disagreement. That disagreement is not our novelty. Our added question concerns sequence structure against the same flattened information and a neural control. |
| [Dinçer and Kilimci (2026 issue)](https://doi.org/10.3390/sym18010032), pp. 7-11 | A shared multivariate sequence pipeline already compares LSTM/Transformers with econometric references for weekly price forecasting. Our comparison adds tabular context and MLP controls for daily direction probabilities. The issue year is 2026; online publication was 24 December 2025. |

[Zeng et al. (2023)](https://doi.org/10.1609/aaai.v37i9.26317) motivate strong simple
baselines in a different, long-horizon forecasting setting; their findings do not
establish cryptocurrency direction results.

The intended applied contribution is the combined history-control, neural-control
and delayed-trading comparison, with training-only duration selection, equal final
samples and dependence-aware uncertainty. It introduces no new learning algorithm
and makes no priority claim. The fixed historical panel, limited training sample,
single profile per model, conditional inference, hypothetical transaction costs
and USDT denomination remain limitations whatever the outcome.
XGBoost is a fixed named comparator, not a test-selected best classical model;
the study does not establish a ranking of every shallow or deep method. In
particular, the SVM results in Kaygın et al. are motivation for strong baselines,
not evidence that our included tabular profiles exhaust that class.
