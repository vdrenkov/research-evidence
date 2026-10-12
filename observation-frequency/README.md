# Observation frequency in cryptocurrency forecasting

## Research question

Does observing the same market history at 1h, 4h or 24h resolution improve
next-day probability forecasts when the forecast horizon, forecast origins,
elapsed lookbacks, training population and feature dimensions are held fixed?

The study provides a controlled comparison using existing models. It uses
historical data that were also examined in earlier analyses. Findings apply to
the selected assets, features and model profiles.

## Data and design

[`study.yaml`](study.yaml) defines the executable settings. All resolutions are
aggregated from the same Binance hourly OHLCV source.

| Component | Specification |
| --- | --- |
| Assets and source period | BTC/USDT, ETH/USDT and SOL/USDT; `[2020-09-01, 2025-09-01)` in UTC. |
| Forecast task | Daily forecast at 00:00 UTC using completed candles; predict the following 24h close-to-close return direction. Zero return is class 0. |
| Test period | Common eligible dates in `[2022-09-01, 2025-08-31)`, shared across assets and resolutions. Six calendar folds; the final fold leaves one day for terminal execution. |
| Training | Previous 24 calendar months per fold. Exclude training examples whose labels end at or after the first test origin. Fit transformations only on eligible training observations. |
| Features | Log return, realized volatility, path efficiency, log high/low range and log total volume over 1, 7 and 28 elapsed days: 15 features per resolution. |
| Controls | Returns, ranges and volumes agree across resolutions on eligible origins. Native-return volatility and path efficiency intentionally vary with resolution. |
| Models | Logistic regression, Random Forest and XGBoost with fixed settings. No test-driven tuning or class weighting. Average probabilities from seeds 17, 42 and 73 for each model/resolution. |
| Benchmarks | Training-only historical up-rate and constant probability 0.5. |

Logistic regression uses `C=1` and training-only standardization. Random Forest
uses 300 trees, depth 6 and minimum leaf size 10. XGBoost uses 250 trees, depth 3,
learning rate 0.05, and row/column subsampling of 0.8. The planned grid has 486 fits.

Synthetic or missing hours invalidate origins whose feature or target windows
touch them. Decision eligibility depends only on past history; scoring also
requires an observed target. Repair flags, exclusions and actual fold counts
are retained. No new filling, interpolation or download occurs. Original provider
retrieval dates are not reconstructed; saved snapshots identify the inputs used.

## Forecast evaluation

The primary measure is Brier score. Six daily-minus-finer contrasts compare
24h with 1h and 4h observations for each model family, with equal asset weights.
Secondary measures are log loss, AUC, accuracy, calibration and skill against
the historical up-rate. All primary contrasts are reported.

Uncertainty uses synchronized circular moving calendar blocks within folds:
5,000 bootstrap replicates, 14-day primary blocks, and 7/28-day sensitivities.
Report ordinary 95% intervals and Bonferroni family intervals for the six
contrasts. These intervals condition on fitted forecasts and the fixed asset
panel, assume local weak dependence and omit model-refitting uncertainty.
Seeds and overlapping training windows are not independent replications.

## Economic evaluation

Daily long/cash decisions use a primary probability threshold of 0.55, with
0.50/0.60 sensitivities. Execution uses the same hourly engine across resolutions,
a one-hour delay after forecast origin, 10,000 USDT initial capital and a 20%
participation cap. Per-fill commission/slippage scenarios are 0/0, 6/5 and 12/10
basis points; 6/5 is primary. The planned grid has 252 policy replays.

Positions continue across folds; incomplete decision histories request cash.
Liquidation is requested after the final holding interval. Saved fills and equity
allow checks of delayed execution and terminal costs. Daily Sharpe uses common
24h valuation marks and is unavailable if a position remains open.

Forecast scores and delayed trading returns measure different outcomes. Frictions
are assumed scenarios rather than measured historical spreads; results do not
establish real-world capacity. USDT depeg risk is not modeled separately.

## Reproduction and evidence

The implementation is maintained in
[Crypto Backtest Framework](https://github.com/vdrenkov/crypto-backtest-framework).
For reproduction, use the recorded software revision and dependency environment,
with clean, committed framework and evidence checkouts. Existing monthly hourly
source partitions including repair flags are required; PostgreSQL is unnecessary.
The launcher currently reads that partition layout; using the committed normalized
snapshots for a fresh full run requires preparing the same input layout.

From the framework checkout:

```powershell
./scripts/run_observation_frequency_study.ps1 -StudyRoot '<absolute path to this study folder>'
```

The command prepares inputs, fits models, replays policies and exports analysis.
Each execution writes a fresh `results/observation-frequency/<run-id>` directory.

### Completed run

[Run `20261012t200333z-25662394`](runs/20261012t200333z-25662394/) completed
486 fits and 252 policy replays using framework release `v3.1.0`. Its
`identity.json` records the exact software and dependency environment, input
hashes and the evidence commit containing the pre-execution design and configuration.

The retained package contains the three input snapshots, forecast checkpoints,
eligibility calendar and identity/completion manifests, copied unchanged from the
original run. Checkpoints retain individual predictions and training records for
verification without refitting. File hashes can be checked against `outputs.json`.

`outputs.json` records hashes for the complete experiment. Excel exports, figures
and trading traces are retained separately in the framework's
`results/observation-frequency/<run-id>/` directory.

| Output | Verification purpose |
| --- | --- |
| `study.yaml`, `design.md`, `identity.json`, `inputs/`, `calendar.parquet` | Frozen study files, software and environment identities, input snapshots, hashes and eligibility calendar. |
| `checkpoints/`, `policies/` | Individual forecasts, fit records, trading fills and equity for checking timing, metrics and accounting. |
| `tables.xlsx`, `forecasts.xlsx`, `figures/` | Consolidated analysis, ensemble/benchmark and individual-seed forecasts, and figures. |
| `outputs.json`, `status.json` | Artifact hashes and execution completion counts. |

These records support reconstruction of reported results. Conclusions remain
limited to the fixed asset panel, tested representations and historical periods.
