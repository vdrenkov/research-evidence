# Cross-market forecasting stability

This retrospective study asks whether the specified sequence models' probability-loss
advantage or disadvantage against history-matched controls changes across selected
cryptocurrency, equity-fund, FX-reference and commodity-fund series. Each system is
trained separately for each instrument and fold. It does not test unchanged-weight
transfer, intrinsic market difficulty or trading profits. Earlier data and comparative
results have already been examined; the new run is not independent confirmation or
public preregistration.

## Inputs and observation times

The fixed panel is BTC/USDT, ETH/USDT, SOL/USDT; SPY, QQQ, IWM; EUR/USD, GBP/USD,
JPY/USD; and GLD, SLV, USO. These are three selected cases per group, not independent
random samples of their markets. USDT and USD are not assumed identical. Fund prices
are not spot commodity prices; ECB rates are informational references, not trades.

`sources.json` identifies the exact retained daily CSV bytes in `inputs/`, original
locations and fund raw-archive identities. `calendars.json` retains independently
constructed U.S. session and ECB publication dates, their intersection and the
seven-day crypto calendar for [2022-09-01, 2026-09-02). The source audit accounts for
U.S. half-days and the exceptional 9 January 2025 closure. Missing expected rows fail
validation rather than disappear through observed-row intersection. Nothing is filled.

Crypto uses the completed UTC daily candle. FX uses the same-date ECB release
(EUR/USD is USD per EUR; GBP/USD and JPY/USD divide the USD reference by the
respective EUR-base reference). Funds use the last observed minute-bar close strictly
inside the scheduled New York core session. Endpoints more than five minutes before
the scheduled close are invalid. Their dependent histories and targets remain excluded.
A common date is a pairing label, not a simultaneous observation time. The input index accesses only its own completed-date and earlier endpoints;
archive adjustments/revisions are not certified point-in-time. There are no cross-series
predictors. Common-calendar targets can span weekends or holidays.

The retained fund archive predates a change in the provider's documentation. Its
archived description says split/dividend-adjusted prices; current documentation says
split-only adjustment and a revised source history. Adjustment factors are not
independently certified. The study therefore concerns these exact provider-archive
endpoints, without claiming official consolidated closes, total returns or historical
point-in-time vintages. Source, adjustment, quote and observation-time differences
remain coupled to the groups. A later download must not silently replace this snapshot.

## Matched forecasting design

The target is 1 when the next expected endpoint price is strictly greater than the
current endpoint and 0 otherwise. Each model receives the same 28 consecutive log
returns, requiring 29 valid price nodes. Tabular inputs flatten the ordered returns;
sequence inputs retain the 28-by-1 array. This short, explicit history avoids combining
long rolling indicators with an additional sequence window. It is a resource-bounded
design choice, not an optimized universal history length.

The main common-calendar models are logistic regression, Random Forest, XGBoost,
MLP, LSTM, GRU, TCN and Transformer. The specified TCN receptive field covers the full
input. The MLP and XGBoost are named non-sequential controls, chosen before this run.
Native-calendar sensitivity separately refits XGBoost, MLP and GRU on each series'
expected native dates, including cryptocurrency weekends. It changes history spacing,
horizon and eligible populations; it is not a causal estimate of a calendar effect.

Four consecutive six-month test folds cover [2024-09-01, 2026-09-01), with rolling
24-calendar-month training windows. History starts again at each training boundary.
The last 90 expected training origins define the validation suffix before quality
exclusions. Labels reaching the validation boundary are purged from selection; labels
reaching the first test origin are purged from final training and validation. Training
and evaluation eligibility match across models and, in the main experiment, instruments.
Native comparisons match models within each instrument. Minimum sample/class support
is checked before fitting; failures stop the run rather than trigger a fallback.

All observations have equal weight. Logistic C is selected from 0.01/0.1/1/10, forest
depth from 3/6 and boosting depth from 2/3. Neural learning rates are 0.001/0.0003.
The first candidate wins a tie in mean-seed validation log loss. Random Forest fits
Gini splits, not a log-loss objective. Boosting and neural duration selection use
validation only; a fresh model is refitted on every eligible outer-training origin.
The neural cap is 100 epochs with patience 12, AdamW, gradient clipping and retained
training/validation curves. Capacity and optimization budgets are not equivalent across
architectures. Exact fixed architectures and parameters are defined in the recorded
framework revision; `study.yaml` declares the selection grids and neural settings.

Seeds are 17/42/73; deterministic logistic regression uses 17 once. Stochastic final
probabilities are averaged before scoring. A stage's scalar standardization counts each
unique return in its training histories once. No class weighting or recalibration is
applied. Constant 0.5 and the corresponding final-training up-rate are unranked references.

## Outcomes and uncertainty

Brier loss is primary. For each group, the sequence-minus-control gap is the equal mean
of LSTM/GRU/TCN/Transformer loss minus the equal mean of XGBoost/MLP loss. The primary
family contains three direct crypto-minus-comparator gap differences. Positive values
mean a less favorable relative sequence result in crypto. Instruments receive equal
weight; valid dates receive equal weight within each fold, and the four folds are equal.
Absolute reference-relative scores are reported alongside these contrasts.

A separate three-comparison secondary family estimates Kendall tau-b agreement between
crypto and each comparator's eight-profile Brier rankings. References are excluded from
ranking. Eight fixed profiles give limited resolution for rank association. The main question concerns probability skill, not winner agreement. Log loss,
AUC, accuracy, reliability, individual instruments/folds and native-calendar results are
descriptive. Native and common-calendar losses must not be pooled as an identical task.

Each family uses 100,000 synchronized circular block draws within the four expected
common-date grids. The primary block is ten observations, with five/twenty-observation
sensitivities (approximately one/two/four common-calendar weeks, not fixed elapsed days).
The shared quality mask is applied after sampling. Percentile intervals and separate
Bonferroni family intervals use alpha 0.05. There is no study-wide error guarantee across
families or sensitivities. Empty-support draws are rejected with counts; undefined rank
draws make the corresponding ranking interval ineligible rather than being silently
removed. Intervals condition on the saved fits and selected panel; they do not represent
retraining, provider-revision or new-instrument uncertainty. Null-crossing intervals do
not establish equivalence or absence of predictability.

## Reproduction and retention

Use the recorded framework revision and the portable `scripts/run_cross_market_stability_study.ps1`
launcher with this folder as `StudyRoot`. Smoke checks every model on the first crypto
training/validation case without scoring outer-test forecasts. Full requires that completed
Smoke and identical clean framework/evidence repositories, dependencies and input bytes.
Each launch records exact commits, environment, configuration, snapshots and run identity
before fitting. Interrupted runs reuse only verified completed cells under that identity.
Completed packages support read-only Verify. No SQL database or trading replay is used.

The design, source identities and compact daily inputs are retained here. Full outputs,
seed predictions, fitted states, training records and XLSX exports live under the
framework's `results/cross-market-stability/<run-id>/`. Preserve that directory and the
original source archives. After verification, retain the necessary compact result evidence
here; models and large minute archives need not enter ordinary Git. Access and any public
redistribution remain subject to source terms and required attribution.

Data sources: [Binance public data](https://github.com/binance/binance-public-data),
[ECB reference rates](https://www.ecb.europa.eu/stats/policy_and_exchange_rates/euro_reference_exchange_rates/html/index.en.html),
and [HF Data Library, Elkassabgi (2026)](https://hfdatalibrary.com/).
Data provided for free by IEX. By accessing or using IEX Historical Data, you agree to the
[IEX Historical Data Terms of Use](https://www.iex.io/legal/hist-data-terms).
The HF compilation/documentation and upstream securities data have distinct rights;
this package does not grant rights beyond their applicable terms.
