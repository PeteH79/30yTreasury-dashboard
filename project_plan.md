# 30-Year Yield Path Dashboard Project Plan

## Objective

Build a professional daily dashboard for evaluating the likely path of the U.S. 30-year Treasury yield. The dashboard should combine refreshed market data, technical charts, model-implied scenario probabilities, source freshness, and a disciplined daily workflow.

## Current Build

- Static local dashboard: `outputs/thirty_year_yield_dashboard/index.html`
- Browser-safe data cache: `outputs/thirty_year_yield_dashboard/data/rates_dashboard_data.json`
- Source refresh script: `tools/refresh_rates_dashboard_data.py`
- Secret location: project-root `.env`, read only by the refresh script

## Data Sources

Core market and macro series are pulled from FRED:

- `DGS30`: 30-year Treasury yield
- `DGS10`: 10-year Treasury yield
- `DGS2`: 2-year Treasury yield
- `DFII30`: 30-year real yield
- `T10YIE`: 10-year breakeven inflation
- `VIXCLS`: VIX
- `BAMLH0A0HYM2`: high-yield option-adjusted spread
- `DCOILWTICO`: WTI crude oil spot

Market proxies are pulled from the Yahoo Finance chart API:

- `^GSPC`: S&P 500
- `^IXIC`: Nasdaq Composite
- `DX-Y.NYB`: U.S. Dollar Index
- `TLT`: long-duration Treasury ETF proxy
- `CL=F`: WTI crude futures proxy

The browser app never receives the FRED API key. It only reads generated non-secret JSON.

FRED Treasury constant-maturity series such as `DGS30` provide one daily yield observation, not open-high-low-close bars. Candlestick charts would need a tradable market proxy with OHLC data, such as Treasury futures (`ZB=F`) or a long-duration ETF/futures proxy from Yahoo or another market-data vendor.

## Refresh Process

Run from the project root:

```powershell
python tools\refresh_rates_dashboard_data.py
```

The script writes:

- `data/rates_dashboard/rates_dashboard_data.json`
- `outputs/thirty_year_yield_dashboard/data/rates_dashboard_data.json`

The dashboard then reads the served output copy.

The refresh script also attempts to pull long-end auction context from TreasuryDirect public web services:

- recent auction results from `TA_WS/securities/auctioned`
- upcoming auctions from `TA_WS/securities/upcoming`

If TreasuryDirect is unavailable, the dashboard remains usable and shows an auction source gap.

Validate the generated data without refreshing:

```powershell
python tools\check_rates_dashboard_data.py
```

For a one-command morning workflow, run:

```powershell
.\tools\start_rates_dashboard.ps1
```

This refreshes the data cache, validates the generated JSON, serves `outputs/thirty_year_yield_dashboard`, and opens the dashboard at:

```text
http://127.0.0.1:8765/index.html
```

Use `.\tools\start_rates_dashboard.ps1 -SkipRefresh` to serve the last cached data without contacting external APIs.

To create a Windows Scheduled Task that refreshes and validates the data cache each morning:

```powershell
.\tools\register_rates_dashboard_task.ps1 -At 07:30
```

Use `-RunNow` to start the task immediately after registration:

```powershell
.\tools\register_rates_dashboard_task.ps1 -At 07:30 -RunNow
```

Scheduled refresh logs are written to:

```text
data\rates_dashboard\logs\scheduled_refresh.log
```

## Probability Model

Current probabilities are transparent heuristic scenario weights, not calibrated investment forecasts.

Scenarios:

- `break_higher`: 30Y yield rises meaningfully
- `range_bound`: 30Y yield remains in range
- `rally_lower`: 30Y yield falls meaningfully

Factor families:

- Technicals
- Inflation
- Real rates
- Curve
- Risk appetite

Each factor is scored from `-2` to `+2`, where positive scores imply higher-yield pressure and negative scores imply lower-yield pressure. The dashboard shows the weighted factor contribution behind the latest probability read.

## Backtest Definition

The current outcome checks are intentionally simple:

- Short horizon: 10 trading days, `+/-15 bp`
- Medium horizon: 20 trading days, `+/-25 bp`
- Slow horizon: 60 trading days, `+/-50 bp`
- Break higher: forward 30Y yield change greater than or equal to the horizon threshold
- Rally lower: forward 30Y yield change less than or equal to the negative horizon threshold
- Range-bound: anything between those thresholds

The dashboard reports the latest 126 evaluable trading days for each horizon, including hit rate, latest evaluable signal, and hit rates by predicted scenario bucket. This is a model diagnostic, not proof of predictive power.

## Daily Workflow

1. Refresh the data cache.
2. Check source freshness and any source gaps.
3. Review the signal trust check for staleness, missing inputs, probability churn, factor conflict, and horizon disagreement.
4. Read the 30Y yield trend and technical panels.
5. Review scenario probability history.
6. Inspect factor contributions to understand what moved.
7. Check catalysts and invalidation rules.
8. Write the morning note.

## Quality Bar

The dashboard should remain:

- Source-backed and refreshable
- Transparent about model logic and data freshness
- Chart-led rather than commentary-led
- Usable on desktop and mobile widths
- Explicit about what would change the yield-path view

## Latest Build Addition

The dashboard now includes:

- Probability change attribution, comparing the latest scenario odds with the prior period.
- Factor contribution history, showing how technicals, inflation, real rates, curve, and risk appetite have pushed the model over the selected daily, weekly, or monthly window.
- Horizon-aware labels so daily, weekly, and monthly views explain the period being compared.
- Multi-horizon calibration diagnostics for 10, 20, and 60 trading days.
- Signal trust checks for source staleness, missing inputs, probability churn, factor conflict, and calibration disagreement.
- A local PowerShell launcher for the daily refresh-and-serve workflow.
- A generated-data health check and Windows Scheduled Task registration helper.
- TreasuryDirect-backed auction watch fields for latest long-end auction results and upcoming long-end supply.

## Next Build Steps

1. Add macro event annotations for CPI, payrolls, FOMC, and refunding/auction dates.
2. Add auction concession/tail estimates by comparing auction high yield with when-issued or nearby market proxies.
3. Consider replacing the heuristic probability model with a calibrated logistic or ensemble model once enough validated features and outcome definitions are stable.
4. Add richer model stability checks, such as factor-score persistence, forecast entropy, and stale-source probability suppressors.
5. Add a lightweight changelog export for daily notes and signal state.
