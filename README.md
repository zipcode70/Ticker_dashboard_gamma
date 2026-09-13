# On-Demand Ticker Gamma Exposure Dashboard

A companion to the SPY daily-tracker dashboard — this one lets you type in
**any** ticker with listed options and get a fresh gamma exposure dashboard
for it, computed on demand from GitHub Actions. This is a separate project
from the SPY dashboard: separate repo, separate automation, separate site.
It does not modify or depend on the SPY dashboard in any way.

## What this shows

For whichever ticker you request, the dashboard computes:

- **Call wall** — the strike with the largest positive call gamma (acts like resistance)
- **Put wall** — the strike with the largest negative put gamma (acts like support)
- **Max pain** — the strike that minimizes total option payout at expiration
- **Gamma flip** — the spot level where modeled dealer gamma crosses from negative to positive
- Full gamma-exposure-by-strike chart, gamma flip curve, and call/put open interest chart

The strike window is auto-sized to \u00b125% around the ticker's spot price, so it
scales correctly whether you check a $50 stock or a $700 index ETF.

**This is a single-snapshot tool, not a tracker.** Each run replaces the
previous ticker's data \u2014 it does not keep a running history across days or
across different tickers (a call wall trend line for AAPL one day and TSLA
the next wouldn't mean anything).

## How to use it

1. Go to the **Actions** tab of this repository.
2. Click **On-Demand Ticker Gamma Exposure** in the left sidebar.
3. Click the **Run workflow** dropdown button.
4. Type the ticker symbol you want (e.g. `AAPL`, `QQQ`, `TSLA`) into the input box.
5. Click the green **Run workflow** button.
6. Wait about a minute for the run to finish (refresh the Actions tab to watch progress).
7. Open your GitHub Pages link for this repo \u2014 the dashboard now reflects that ticker.

Repeat any time you want to check a different ticker. There's no schedule \u2014
it only runs when you trigger it.

## Setup (one-time)

Same setup steps as the SPY dashboard, applied to this separate repo:

1. Create a new GitHub repository and upload all the files in this project
   (keep the folder structure, including the hidden `.github/workflows/` folder).
2. **Settings \u2192 Actions \u2192 General \u2192 Workflow permissions** \u2192 select
   "Read and write permissions" \u2192 Save.
3. **Settings \u2192 Pages \u2192 Source** \u2192 select "GitHub Actions".
4. Go to the **Actions** tab, run the workflow once manually with a ticker of
   your choice to generate the first dashboard and confirm everything works.

## Project structure

```
.
├── scripts/
│   └── compute_gex.py          # takes a ticker (env var or --ticker flag), computes GEX
├── site/
│   ├── index.html
│   ├── styles.css
│   ├── app.js
│   └── data/
│       └── gex_data.json       # current snapshot for whichever ticker was last run
├── .github/workflows/
│   └── on-demand-gex.yml       # workflow_dispatch only, takes a "ticker" input
├── requirements.txt
└── HANDOVER.md                 # architecture notes for handing this off to another AI tool
```

## Notes

- Some tickers don't have listed options (or don't have any within the
  auto-sized strike window) \u2014 the run will fail with a clear error message
  in the Actions log if so.
- Data source: [Yahoo Finance via `yfinance`](https://pypi.org/project/yfinance/).
  This is a free, unofficial source, so occasional stale reads or minor
  quote lag are expected.
- Risk-free rate is fetched live each run from the 13-week T-bill (`^IRX`),
  with an automatic fallback to a hardcoded 3.9% if that fetch fails.
