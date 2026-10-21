# Additional information and availability delays in cryptocurrency forecasting

This study tests whether selected on-chain activity, crypto-peer and Wikimedia
attention features improve next-day direction probabilities beyond market
features, and whether the result changes under longer assumed on-chain delays.
`study.yaml` specifies execution settings; `sources.json` identifies the exact
existing source snapshots and original response files. Earlier historical data,
results and reviews informed this design. The new experiment is retrospective,
not independent confirmation or preregistration.

## Research question and contribution

Existing work already studies on-chain predictors, peer predictability and
multimodal cryptocurrency forecasting. Kim et al. test on-chain variable groups
([2022](https://doi.org/10.1109/ACCESS.2022.3177888)); Guo et al. study
cross-cryptocurrency returns
([2024](https://doi.org/10.1016/j.jedc.2024.104863)); Gurgul et al. combine financial,
blockchain and social inputs and compare added NLP information
([2025](https://doi.org/10.1016/j.ijforecast.2025.02.007), available
[author preprint](https://arxiv.org/abs/2311.14759)). These studies motivate the
blocks; adding more data or models alone is not a new contribution.

The bounded contribution is a matched information-block comparison under explicit
availability assumptions. It separates feature-set changes from changed date
populations and examines whether conclusions survive 24-, 48- and 72-hour
on-chain delays. It does not identify every useful blockchain signal, causal
information value, historical provider vintages or optimal model-family performance.

## Inputs and timing

Use Binance BTC/USDT, ETH/USDT and SOL/USDT hourly snapshots over
`[2020-09-01, 2025-09-01)`. Forecast origins are 00:00 UTC on
`[2022-09-01, 2025-08-31)`, ending 30 August 2025. The label is one if the close
of the next completed UTC day exceeds the last observed daily close; zero returns
belong to class zero. At origin `t`, the latest price input is the previous
23:00 hourly close. These labels and trading returns use different intervals.

The retained Coin Metrics snapshot contains BTC/ETH `AdrActCnt`, `TxCnt`,
`TxTfrCnt` and `FeeTotNtv`. Wikimedia inputs are English article pageviews,
`all-access`, `user`, for Bitcoin, Ethereum and `Solana_(blockchain_platform)`.
The original responses were downloaded on 6 August 2026; measurement histories
end in March 2026. Only the declared earlier study window is evaluated. SOL
attention begins on 10 September 2021, reducing its first training population.
Absence before article coverage is not zero attention. Alternative.me is excluded.

For on-chain measurements ending at `u`, scenario availability is `u + 24h`,
`u + 48h` or `u + 72h`. Attention always uses `u + 24h`. Peer features use only
completed price bars. Backward joins require an observation at the expected daily
release time: no stale-day carry, interpolation, backward filling or fabricated
zero values. Original timestamps remain unchanged; scenario timestamps and source
lineage are retained separately. These assumed delays cannot remove later
provider revisions or establish point-in-time historical availability.

## Feature arms and matched populations

- **T, market:** returns over 1/3/6/12 days; return volatility over 12/24/48 days;
  RSI(14); EMA(12)/EMA(26) minus one; EMA(20)/EMA(50) minus one;
  daily high-low range divided by close; volume z-scores over 24/72 days.
- **X, peers:** the other two assets' 1/3-day returns, their equal-weight one-day
  return and positive-return breadth. The target asset is excluded.
- **O, on-chain:** one-day active-address change and its 30-day z-score,
  transaction-count, transfer-count and native-fee-total one-day changes.
- **S, attention:** one-day pageview change and its 30-day z-score. This measures
  attention, not sentiment.

BTC/ETH use T, TX, TO, TS, TXO, TXS, TOS and TXOS. O-containing arms are fitted
separately at each delay; unchanged arms are fitted once. SOL uses T, TX, TS and
TXS as a separately refitted secondary scope, without on-chain or transfer claims.
There are 36 distinct asset/arm/delay combinations.

Technical indicators are recomputed from exactly 73 completed daily bars, including
finite-window EMA initialization. This covers the longest registered 72-day volume
window and bounds repair dependencies; it is a new market baseline, not the rebuilt
Paper 3 input representation. RSI uses exponentially weighted gains/losses with
alpha 1/14, returning 100 for positive gains with no losses and 50 for a flat window.
Volatility uses sample standard deviation; z-scores use population standard deviation.
A repaired hour invalidates its daily bar and every
dependent technical window or target. Unavailable external features invalidate
their dependent transformations. Core BTC/ETH training and decision dates are the
intersection across both assets, every arm and all delays; scoring also requires
both observed targets. SOL has its own all-arm intersection. Decisions use past
inputs only. The launcher reports actual support and exclusions before fitting;
it stops on inadequate support rather than silently changing the design.

## Validation and training

Six rolling folds use 24 calendar months for training and six for testing,
truncating the last fold at the declared end. The last 90 training calendar days
form validation. Selection-training labels must end strictly before validation
starts; final-training and validation labels must end strictly before testing.
At least 150 selection, 60 validation and 250 final training examples are required,
with both classes in every stage; each asset/fold needs 60 scored test dates.
The selection minimum was set after a pre-execution coverage/repair audit, not
model outcomes: SOL's first selection window has at most 172 usable examples,
versus 225 for the core assets. Its limited early sample is a secondary-scope
limitation; actual feature-complete counts are retained by execution.

The models use unweighted observations. Logistic regression and XGBoost optimize
log loss; Random Forest uses Gini splits and log-loss selection. This clarifies
the frozen description’s broader objective wording; the executed code and results
are unchanged. Logistic regression selects C from
0.01/0.1/1/10, uses training-only standardization and at most 3,000 iterations.
Random Forest selects depth 3/6, with 300 trees and minimum leaf size 10.
XGBoost selects depth 2/3, learning rate 0.05, maximum 250 rounds, subsample and
column fraction 0.8, lambda 1 and 20-round validation early stopping.
Choose the candidate with lowest mean-seed validation log loss; ties use declared
order. Forest and XGBoost use seeds 17/42/73; deterministic logistic regression is
fitted once. Refit from scratch on the full eligible training window; each XGBoost
seed uses its selected duration. Average stochastic seed probabilities before
evaluation. Retain explored candidates, selected durations, training/validation
scores, XGBoost round curves, training populations and final fitted states. There
is no test-based selection, class weighting or probability recalibration.

The full design contains **3,456 selection fits and 1,512 final fits**.
Constant 0.5 and the same eligible final-training up-rate provide probability
references. Six overlapping training windows, correlated assets and repeated seeds
are not independent replications.

## Outcomes and uncertainty

Brier loss is primary; negative augmented-minus-reference differences favour the
augmentation. Average daily paired losses equally over the three model profiles
and then equally over BTC/ETH. Four primary comparisons are TX/T, TO/T, TS/T and
TXOS/T at 24 hours, forming one Bonferroni family. Three separate secondary families
cover TXOS versus TOS/TXS/TXO (three comparisons); each O-containing arm at 48/72
versus 24 hours (eight); and SOL TX/TS/TXS versus T (three).

Use 100,000 synchronized circular calendar-block draws within folds, with 14 days
preferred and 7/28-day sensitivities. Sample the full calendar before applying the
shared score mask. Report ordinary 95% and family-adjusted percentile intervals,
family sizes, valid support and simulation tail counts. Empty draws are rejected
and counted. Each family has its own nominal 5% control under the approximation;
there is no overall 5% guarantee across families or block lengths. Intervals are
conditional on fixed forecasts and omit refitting and provider-vintage uncertainty.
Block-length sensitivity does not validate coverage; null-crossing intervals are
not equivalence. No uncentered bootstrap tail counts are called p-values.

Report absolute Brier loss and skill against both references, log loss, AUC,
accuracy, probability spread and fixed ten-bin reliability descriptively. Keep
model-specific and fold-specific results visible without selecting favourable cells.

## Descriptive trading and retained evidence

Replay every learned arm/scenario plus buy-and-hold and cash: **342 cases** across
commission/slippage costs of 0/0, 6/5 and 12/10 basis points per side. The fixed
long/cash threshold is 0.55. First execution is one hour after the origin; holdings
cover `[t + 1h, t + 25h)`. Carry positions across folds and charge terminal
liquidation. Missing decision histories imply cash. Repaired prices cannot supply
fills or required daily marks; other flagged intraday marks are carried for
accounting only. `max_participation=1.0` disables the engine's volume cap;
volume-sensitive slippage is zero. These are unconstrained bar-price fills,
not a causal liquidity model. Retain descriptive requested-volume participation,
zero-volume fills and any incomplete requests before interpreting economic results.
Report exposure, turnover, fees, return, daily Sharpe and drawdown together.
These outcomes do not establish timing skill or deployability.

The framework launcher is `scripts/run_information_availability_study.ps1`.
Smoke checks the complete source/preparation path and reduced-size model fitting
within the first training window; it computes no outer-test scores. Full requires
the successful Smoke with identical committed repositories, configuration,
environment, hardware and input bytes. Keep both repositories unchanged through
completion or resume. Commands are in the framework's `docs/research.md`.

Each fresh framework `results/information-availability/<run-id>/` directory archives
the design, source responses, normalized observations, hourly inputs, aligned
features, eligibility, environment, checkpoints, forecasts, fitted states and
policy traces. It exports one dataset and one analyses workbook. Failures and
identity violations remain recorded; completed artifacts are not overwritten.
The verified completed run is `20261020t175611z-ad040253` (framework `69d83d5`,
design commit `73286ad`): 3,456 selection fits, 1,512 final fits and 342 policy
replays. Exact verification copies are retained in `runs/20261020t175611z-ad040253/`,
including inputs, aligned features, forecasts, training records and the analysis
workbook. The original manifest describes the full run. Detailed fitted models,
policy traces and the original dataset export remain in the framework result
store at the corresponding manifest paths.

Repositories remain private; reviewer access can be arranged subject to provider
conditions. Wikimedia API data are CC0 under its
[access policy](https://doc.wikimedia.org/generated-data-platform/aqs/analytics-api/documentation/access-policy.html).
Coin Metrics describes Community API use as noncommercial under a Creative Commons
licence in its [official documentation](https://github.com/coinmetrics/product-docs/blob/master/docs/access-our-data/api/README.md).
Retain attribution and the original source identities. Public redistribution of
network data requires confirmation of the applicable terms; the software licence
does not license source datasets. No public release or new acquisition is part of
this execution.
