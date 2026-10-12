# Kalman-Filtered Pairs Trading with Rolling Coherence Gating

A research notebook comparing a baseline Kalman-filtered pairs-trading rule with a coherence-gated variant across a predeclared equity-pair universe.

> **Research disclaimer:** this is an exploratory backtest, not investment advice or a production trading system. The configured 2023–2025 evaluation period has informed previous research decisions and is **not an untouched holdout**.

## Project overview

The notebook implements two systems:

- **Normal Kalman:** a causal filter estimates the hedge ratio from log prices and the trading rules act on a normalized spread.
- **Coherence-gated Kalman:** the same core framework uses squared rolling Pearson correlation of the two assets' returns as a coherence proxy, with model-update gating and configured trade-level hedge-ratio / z-score behavior.

The coherence measure is a simple rolling-return-correlation proxy, **not wavelet coherence**. The project evaluates multiple pairs, chooses parameters on development data, and reports per-pair metrics, combined summaries, bootstrap intervals, and plots.

## Notebook

- `kalman_coherence_pairs.ipynb` — complete analysis, synthetic-data regression tests, market-data loading, model selection, backtest accounting, diagnostics, and exports.

## Setup

Python 3.10+ is recommended. Install the required packages in your environment:

```bash
python -m pip install numpy pandas matplotlib scipy yfinance jupyter
```

Open Jupyter and run `kalman_coherence_pairs.ipynb` from top to bottom.

## Configure before running

The first code cell contains the `CONFIG` dictionary. Review these fields before starting a run:

| Setting | Default | Purpose |
|---|---|---|
| `PAIRS` | 15 predefined equity pairs | Pair universe; freeze it before evaluating results. |
| `DATA_SOURCE` | `download` | Download adjusted close prices with `yfinance`, or use `csv`. |
| `START` | `2015-01-01` | Start of required price history. |
| `CALIBRATION_END` | `2020-01-01` | Exclusive end of threshold-calibration history. |
| `TRAIN_BLOCKS` | `2020-01-01` to `2023-01-01` | Continuous development-selection interval and diagnostic block(s). |
| `TEST_START`, `TEST_END` | `2023-01-01`, `2026-01-01` | Evaluation window; `TEST_END` is exclusive. |
| `STOP_GRID` | 3.8, 4.0, 4.2, 4.4, 4.6 | Candidate stop thresholds. |
| `LOOKBACK_GRID` | 10, 15, …, 50 | Coherence lookbacks searched by the proposed method. |
| `COST_BPS` | 5.0 | Assumed cost per turnover unit, in basis points. |
| `SHOW_EACH_PAIR` | `False` | Set `True` to render pair-level heatmaps, ledgers, and trade diagnostics. |

`TEST_ALREADY_INSPECTED` defaults to `True` deliberately. Do not set it to `False` unless both the evaluation period and pair universe genuinely have not influenced your choices.

### Using cached CSVs

Set `DATA_SOURCE` to `csv` and place one file per pair in `cache/` using the name `{Y}_{X}_prices.csv`. Each file must contain exactly the relevant columns `Date`, `{Y}`, and `{X}` (additional columns are ignored). Dates should cover the configured start and end. Example for `AAPL/MSFT`: `cache/AAPL_MSFT_prices.csv` with columns `Date,AAPL,MSFT`.

When using `download`, prices are cached locally as well. Downloaded historical prices can change due to provider revisions, so retain the actual cache files if strict reproducibility is important. They are intentionally not included in this repo.

## Method and evaluation design

1. Prices are aligned by date, transformed to log prices for hedge-ratio estimation, and simple returns are retained for accounting.
2. Calibration thresholds use observations before `CALIBRATION_END`.
3. Every candidate is scored on the continuous development ledger using net annualized Sharpe, subject to the minimum total trade count. Ties prefer a lower stop and then shorter lookback.
4. The selected configurations are evaluated without a second parameter sweep on the evaluation period.
5. Signals determined at close *t* execute at close *t+1*. The accounting ledger includes the configured transaction-cost assumption and drift/rebalancing turnover.
6. Synthetic-data checks test causality and timing invariants. They are implementation checks, not evidence of market profitability.

The proposed method has more candidates than Normal Kalman because it searches stop threshold × coherence lookback. This search-budget imbalance increases its opportunity to overfit and should be considered in interpretation.

## Outputs

The notebook creates `final_multiple_pair_outputs/` with:

- the experiment protocol and selected-parameter manifests;
- per-pair candidate grids, evaluation metrics, daily ledgers, trade logs, and paired intervals;
- a list of failed or ineligible pairs;
- cross-pair comparison tables and aggregate daily returns / metrics;
- saved PNG plots for performance, candidate grids, and diagnostic relationships.

The equal-weight aggregate is built from the intersection of dates for pairs that completed both runs. It is a comparative research summary, not a fully constrained capital-allocation simulation; pairs may be correlated and may share securities.

## Limitations

- **No untouched holdout in the current defaults.** The `2023-01-01`–`2026-01-01` interval covers 2023–2025 and is labeled exploratory because it influenced earlier research and implementation choices. Confirm on new data or a predeclared untouched universe.
- **Multiple testing / selection bias.** A fixed grid helps make decisions auditable but does not remove data snooping. The method with more candidates has a larger search budget.
- **Mechanism attribution.** The proposed system changes multiple linked rules. A paired performance difference does not isolate the effect of coherence gating alone.
- **Execution realism.** Adjusted close prices are only a proxy. Borrow constraints and fees, funding, taxes, market impact, and intraday execution are not fully modeled. Daily stops may gap through thresholds.
- **Risk interpretation.** A spread z-score is not necessarily Gaussian; a z-score stop is not a maximum account drawdown.
- **Statistical uncertainty.** Moving-block bootstrap intervals are conditional on the selected configurations and do not fully correct for model search or cross-pair dependence.

For a discussion of backtest overfitting, see Bailey et al., [*The Probability of Backtest Overfitting*](https://www.davidhbailey.com/dhbpapers/backtest-prob.pdf). The notebook does not implement the paper's PBO estimator.

## Suggested repository hygiene

Keep cached price files, generated CSV/JSON outputs, notebook checkpoints, and virtual environments out of version control. Add a `.gitignore` containing at least:

```gitignore
cache/
final_multiple_pair_outputs/
.ipynb_checkpoints/
__pycache__/
.venv/
venv/
```

No market-data cache or generated result is required to read the notebook source.
