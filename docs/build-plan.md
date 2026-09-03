# Storely Quant — Build Plan
*A multi-horizon, multi-source AI + quant trading research system*

> Not financial advice. This is a technical/educational architecture plan. Have a compliance-literate professional review anything before real capital is involved.

---

## 0a. Locked-in constraints (as of this planning session)

- **Capital target:** Under ₹5 lakh eventually deployed. Data/infra spend must stay near-zero until the system has a real track record — free tiers only, no paid alt-data.
- **Build ownership:** Claude writes the code, user directs/reviews. Build happens in Claude Code against a persistent repo, not in this chat.
- **Time commitment:** Variable — the plan is structured in checkpointed phases so gaps between sessions don't lose progress.
- **Leverage:** None to start. Cash equities (delivery + possibly intraday-cash) only. No F&O until there's a track record and a reason to take on that complexity/tax treatment.
- **Position sizing reality check:** At ₹5L with 2-3% max position size, that's ₹10-15k/trade — brokerage + STT + slippage will eat a meaningfully larger % of small trades than the same strategy would show at ₹50L+. The cost model has to be conservative, not optimistic.

## 0. Ground rules before a single line of code

1. **No live capital until paper trading clears a statistical bar.** Minimum: 3 months of paper trading, positive risk-adjusted return, drawdown within your defined limit, and results that survive a walk-forward test on data the model never saw.
2. **India-specific compliance (applies now, not "someday"):**
   - SEBI's retail algo framework is fully mandatory since April 1, 2026.
   - Under 10 orders/second → no separate exchange registration needed, but your broker still tags your strategy with a Strategy ID and monitors it.
   - API access requires: a static IP whitelisted with your broker, OAuth-only login, mandatory 2FA per session, and sessions auto-close daily — build your system to reconnect/re-auth cleanly, not run unattended for days.
   - If you ever plan to offer the strategy to *other* people (not just your own account), that triggers Research Analyst registration requirements — different project entirely, flag it if it comes up.
3. **Hard risk limits, defined before backtesting starts, not after:**
   - Max drawdown that triggers an automatic kill-switch (e.g. -15% of paper/live capital)
   - Max position size per trade (e.g. 2-5% of capital)
   - Max daily loss limit
   - These are config values enforced in code, not "rules I'll remember to follow."
4. **Every model must beat a dumb baseline** (buy-and-hold Nifty/S&P, or a simple moving-average crossover) net of realistic transaction costs, or it doesn't graduate to paper trading.

---

## 1. Environment & Stack

| Layer | Tooling |
|---|---|
| Language | Python 3.11+ |
| Data handling | pandas, polars (for speed on large datasets) |
| ML (tabular) | LightGBM, XGBoost, scikit-learn |
| ML (sequence) | PyTorch, Temporal Fusion Transformer (via `pytorch-forecasting`) |
| NLP / sentiment | Claude or GPT API for summarization & event extraction; FinBERT for finance-specific sentiment scoring |
| Backtesting (fast/vectorized) | VectorBT |
| Backtesting (event-driven, realistic) | NautilusTrader or Backtrader |
| Broker/data (India) | Zerodha Kite Connect API, NSE/BSE historical data |
| Broker/data (global, optional) | Alpaca (US equities, has a proper paper trading sandbox), Polygon.io, Financial Modeling Prep |
| Experiment tracking | MLflow or Weights & Biases |
| Infra | A cheap VPS with a **static IP** (required for the Kite API), or your own machine with a static IP registered

---

## 2. Data Sources by Domain (with realistic cost tiers)

**Technical (free/cheap):** yfinance, Kite historical API, Alpha Vantage free tier.

**Fundamentals (free/cheap):** SEC EDGAR (US filings, free), Screener.in (India, scrapeable with permission/API), Financial Modeling Prep.

**News & sentiment (cheap-moderate):** NewsAPI, GDELT (free, huge), RSS feeds from Moneycontrol/ET/Reuters, then run through an LLM for entity extraction and sentiment scoring rather than keyword matching.

**Alt/satellite data (be realistic — this tier is expensive):** Orbital Insight, SpaceKnow, Advan Research — these are enterprise products, often $10k+/year minimum. **Out of scope entirely at ₹5L capital.** If you want an alt-data flavor cheaply, Google Trends and app-download-rank data (via public APIs) give a taste of the same "real-world activity" signal for near-zero cost. Revisit satellite/enterprise alt-data only if capital scales up meaningfully later.

**Macro:** FRED (Federal Reserve, free), RBI database (India, free).

---

## 3. Time Horizons → Models → Strategies

| Horizon | Primary data | Model type | Strategy style |
|---|---|---|---|
| Intraday | Technicals, order flow, news sentiment momentum | Gradient boosting classifiers on short-window returns | Statistical arbitrage / mean reversion (be honest: retail intraday edge is thin after costs) |
| Days–weeks (swing) | Technicals + news catalysts | Gradient boosting + rule-based breakout filters | Momentum / breakout, mean-reversion on overreaction |
| 1–3 years | Fundamentals + macro | Factor scoring (Value/Quality/Growth) + gradient boosting on forward returns | Systematic factor investing |

Each horizon is a **separate model with a separate validation protocol** — don't try to build one model that does everything.

---

## 4. Backtesting Discipline (this is where most retail systems quietly lie to themselves)

1. **Vectorized pass first** (VectorBT) — sweep thousands of parameter combos fast to find anything promising.
2. **Purged & embargoed walk-forward validation** — never plain k-fold on time series. Train on period A, embargo a gap, test on period B, roll forward.
3. **Event-driven simulation** (NautilusTrader/Backtrader) on the survivors — this models order fills, latency, and bid-ask spread instead of assuming perfect execution.
4. **Transaction cost stress test** — apply real Zerodha/broker fees, STT, slippage assumptions, and see if the edge survives. Most don't.
5. **Monte Carlo on trade sequence** — reshuffle trade order thousands of times to see the range of possible drawdowns, not just the one path you got.
6. **Overfitting check** — if a strategy needs more than a handful of parameters to work, be suspicious of it. Simpler models that survive out-of-sample beat complex ones that only worked in-sample.

---

## 5. Paper Trading + Daily Post-Mortem Loop

**Paper trading setup:** Zerodha doesn't have a native paper-trading sandbox for algo use the way Alpaca does — for India equities, the practical option is running your model's signals against live market data feed and simulating fills yourself (log the theoretical entry/exit, compare to what actually happened on the chart). Alpaca's paper environment is good to practice the *pipeline* even if you end up trading India markets live later.

**Daily post-mortem — build this as a script that runs every evening:**
1. Pull the day's actual OHLCV/candle chart for every symbol you traded or signaled on.
2. Overlay your model's theoretical entry/exit against the actual chart.
3. Compute: slippage (theoretical fill vs. simulated actual fill), and P&L attribution.
4. Tag every trade with a root cause: `false breakout` / `news event not priced in` / `macro regime shift` / `model correct but sized wrong` / `missed entirely — signal came too late`.
5. Weekly rollup: is the win rate/edge decaying over time, or was this just noise? This is the difference between "the model is broken" and "we had a bad week."

This loop is arguably the single highest-value piece of the whole system — it's what turns "trial and error" into actual learning instead of just error.

---

## 6. Where "AI" (LLMs) actually adds value vs. hype

- **Good use:** LLMs summarizing news/filings into structured sentiment/event tags, extracting "what changed" from an earnings call transcript, writing your daily post-mortem report in plain English.
- **Good use:** Gradient boosting (LightGBM/XGBoost) fusing many weak signals (technical + fundamental + sentiment scores) into a ranked prediction — this is genuinely where most of the "quant AI" edge in practice comes from, not deep learning hype.
- **Use with caution:** Temporal Fusion Transformers / deep sequence models — powerful, but need a lot of clean data to not just memorize noise. Don't reach for these until the simpler models are working and you understand *why* they work.
- **Don't do:** treat any model as an oracle that "predicts the stock price." You're building a probabilistic edge-detector, not a crystal ball. Every model output should come with a confidence/uncertainty measure, not a single number.

---

## 7. Executable next steps (first 2 weeks)

1. Set up the environment (Python stack above), get a Zerodha Kite Connect developer account and a static IP (VPS is easiest).
2. Pull 5+ years of historical OHLCV for a small universe (20-30 liquid NSE stocks) via yfinance/Kite — get comfortable with the data before touching models.
3. Build one dumb baseline strategy (e.g., 20/50 moving average crossover) end-to-end through the full pipeline: backtest → cost model → paper trade → post-mortem script. This proves the pipeline works before any "AI" enters the picture.
4. Only after step 3 works, swap the baseline signal for your first gradient boosting model on technical features, and re-run the same pipeline.

---

## Open questions still to lock down

- **Asset scope:** India equities only (default assumption given location + capital size), or do you also want US equities/crypto/forex exposure later? (Forex/crypto access from India has its own RBI/FEMA constraints worth checking separately — not recommended as a starting point at this capital size regardless.)
- **Broker:** Confirm Zerodha (Kite Connect) as the broker, since that's the most common India retail algo-friendly API. Do you already have a Zerodha account, or does that need setting up?
- **Repo/build environment:** Set up a GitHub repo + Claude Code for the actual build, or keep everything local for now?
- **Starting model:** Dumb baseline pipeline first (recommended — proves the whole pipeline end-to-end before any "AI" enters), or dive into a specific model first?
