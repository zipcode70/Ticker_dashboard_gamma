# Handover Notes — On-Demand Ticker Gamma Exposure Dashboard

This file exists so you (a different AI tool \u2014 Codex, Gemini, Claude Code,
etc., or a future session of any of them) can pick up work on this project
without re-deriving the design from scratch. Written for a non-coder owner,
so it favors plain explanations over jargon.

## What this project is, in one paragraph

A static site (`site/`) plus a Python script (`scripts/compute_gex.py`) that,
given any ticker with listed options, computes a dealer gamma exposure (GEX)
snapshot and renders it as an interactive dashboard. It is triggered manually
from the GitHub Actions tab (`workflow_dispatch` with a `ticker` text input)
\u2014 there is no schedule. Each run overwrites the single data file with that
ticker's results and republishes the site to GitHub Pages.

**This is a sibling project to a separate SPY-only daily-tracker dashboard**
(different repo). They share no code or data at runtime; this one was
generalized from that one but deliberately decoupled so changes here can
never affect the SPY dashboard or vice versa.

## Why this is single-snapshot, not multi-ticker/history

The original SPY dashboard keeps a `history.json` that appends one row per
trading day, so you can watch the call wall / gamma flip drift over the week
for the *same* underlying. That only makes sense for one fixed ticker.

This project trades that away on purpose: since the whole point is to check
*different* tickers on demand, a shared history file would either (a) mix
incomparable data from different underlyings on one trend line, or (b) need
a per-ticker history file scheme with a manifest, dropdown selector, etc.
That's a reasonable v2 if ever wanted (see "Natural next enhancements" below)
but was deliberately left out to keep v1 simple and easy to reason about.

## Pipeline (`scripts/compute_gex.py`)

1. **Ticker resolution:** `--ticker` CLI flag \u2192 `TICKER` env var (set by the
   GitHub Actions input) \u2192 default `"SPY"`. Always uppercased.
2. **Spot price + as-of date:** `get_spot_and_asof(ticker)` pulls the last 5
   days of daily closes via `yfinance` and uses the most recent close. On
   weekends/holidays this naturally reflects the last trading session.
3. **Risk-free rate:** `get_risk_free_rate()` fetches the live 13-week T-bill
   yield (`^IRX`), converts from percentage points to a decimal, and
   sanity-checks it's between 0% and 20%. Falls back to a hardcoded 3.9% on
   any failure. The snapshot JSON records which path was used via
   `risk_free_rate_source` (`"live"` or `"fallback"`).
4. **Auto strike window:** `auto_strike_range(spot, window_pct=0.25)` sets
   `strike_low`/`strike_high` to \u00b125% around spot. This is the key
   generalization vs. the SPY-only version, which had hardcoded
   `STRIKE_LOW = 700` / `STRIKE_HIGH = 850` tuned to SPY's price level only.
   Adjustable via `--window-pct` for local runs.
5. **Option chain:** `fetch_chain()` pulls every expiration within 100 days,
   filters to the auto strike window, and raises a clear `RuntimeError` if
   the ticker has no listed options or nothing falls inside the window (this
   surfaces as a readable failure message in the GitHub Actions log).
6. **Gamma model:** standard Black-Scholes gamma (`bs_gamma`), same formula
   as the SPY version.
7. **GEX formula per contract:**
   `gamma * openInterest * 100 (contract size) * spot^2 * 0.01`, sign
   positive for calls, negative for puts (standard dealer-short convention).
8. **Call wall / put wall:** strike with max aggregated call GEX / min
   aggregated put GEX.
9. **Max pain:** strike minimizing aggregate option payout at expiry across
   included expirations.
10. **Gamma flip curve:** total dealer GEX evaluated across a hypothetical
    spot grid spanning the strike window. Unlike the SPY version (fixed $1
    step), this version uses `np.linspace(strike_low, strike_high, 150)` \u2014
    a fixed *number* of points rather than a fixed dollar step, so resolution
    stays sensible whether the strike window is a few dollars wide (e.g. a
    $20 stock) or hundreds of dollars wide (e.g. a $2,000+ stock).
11. **Output:** writes the full snapshot (including `ticker`,
    `strike_window_pct`, `risk_free_rate`, `risk_free_rate_source`) to
    `site/data/gex_data.json`. No history file.

## Frontend (`site/`)

- `index.html` / `app.js` / `styles.css` \u2014 same visual design language as
  the SPY dashboard (dark theme, Chart.js 4.4.4 + chartjs-plugin-annotation
  3.0.1), but every ticker-specific label (`<title>`, header `<h1>`, KPI
  label, axis titles, chart descriptions) is now set dynamically in
  `renderKPIs()` from `data.ticker` rather than hardcoded as "SPY".
- The daily wall tracker chart/table from the SPY version was removed
  entirely (no history data to show).
- `app.js`'s `init()` catches the fetch failure that occurs before the first
  workflow run (no `gex_data.json` exists yet) and shows a friendly
  "How to check a different ticker" panel with the exact Actions-tab steps,
  instead of a raw error.
- OI chart bucket size is now computed dynamically as roughly
  `(strike range width) / 20`, snapped to the nearest $0.50, instead of a
  fixed $5 bucket \u2014 needed since strike windows can be very narrow or very
  wide depending on the ticker's price.

## Automation (`.github/workflows/on-demand-gex.yml`)

- `workflow_dispatch` only, with a required `ticker` text input (default
  `"SPY"`). No `schedule:` trigger \u2014 this only runs when someone clicks
  "Run workflow" and types a ticker.
- Same two-job shape as the SPY workflow (`update-data` then
  `deploy-pages`), but commits only `site/data/gex_data.json` (no
  `history.json`).
- `concurrency.group: gex-update` prevents two runs from racing each other's
  commits if you happen to trigger it twice quickly.

## Known issues / things to watch

1. **Not all tickers have listed options**, or may have thinly-traded chains
   with mostly zero open interest \u2014 the dashboard will still render but the
   walls/max-pain may not be meaningful for illiquid names. Consider adding
   a minimum-open-interest sanity check if this becomes an issue.
2. **`yfinance` is unofficial** and can occasionally return incomplete or
   stale data, or rate-limit under heavy use. If a run fails intermittently,
   it's usually transient \u2014 just re-run the workflow.
3. **±25% strike window is a fixed default** \u2014 for very high-volatility
   names this might not capture the full relevant chain, and for very
   low-volatility names it might include far more strikes than useful. The
   `--window-pct` flag exists for local tuning but isn't exposed as a
   workflow input yet (see enhancements below).
4. **No data-quality guardrails** beyond "does this ticker have any options
   in the window" \u2014 unlike the SPY version, this hasn't been checked against
   a long baseline of known-good runs, since every run is a different
   underlying.

## Natural next enhancements (if asked "what should we add")

- **Expose `window-pct` as a second workflow input** so you can widen/narrow
  the strike window per run without editing code.
- **Per-ticker history**: write to `site/data/history/<TICKER>.json` instead
  of a single shared file, plus a small `manifest.json` listing which
  tickers have been checked before, so a dropdown could let you flip between
  previously-computed tickers instead of always showing just the latest one.
- **Minimum open interest / minimum volume filter** to warn when a ticker's
  chain is too thin for the walls to be meaningful.
- **Second underlying comparison view** \u2014 run two tickers in one workflow
  dispatch and show them side by side.
