# Storely Quant — Agent Architecture Blueprint

Companion document to `storely-quant-build-plan.md`. This answers: what are we building, how does each piece help, why this design over the alternatives, and where overfitting gets stopped.

---

## Why a multi-agent design at all (the alternative, and why it loses)

**Alternative considered:** one monolithic script/model that ingests everything and spits out buy/sell signals.

**Why we're not doing that:** a single black box is impossible to debug when it's wrong — you can't tell if the loss was a bad prediction, bad sizing, or bad execution. It's also impossible to test in isolation, which is exactly how overfitting hides. Separating concerns into agents means each one has its own inputs, outputs, and — critically — its own validation test that has to pass *before* it's allowed to hand data to the next agent.

**Best alternative, chosen:** a pipeline of narrow, single-responsibility agents, each independently testable, with hard gates between them (a signal doesn't reach execution unless it clears the risk agent's checks). This is standard practice in real quant shops for exactly this reason — not because "agents" is a trendy word.

---

## The seven agents

### 1. Research agent
**Job:** ingest raw data (prices, fundamentals, filings, news) and turn it into structured, tagged information the rest of the system can use — not predictions, just organized facts.
**Inputs:** market data feeds, SEC/BSE-NSE filings, news APIs, the nightly global scan (below).
**Outputs:** a clean, timestamped dataset with entities tagged (which stock, what happened, when) and sentiment/event labels attached to news.
**Why separate from prediction:** if research and prediction are the same step, a bug in data cleaning silently becomes a "trading signal" instead of an obvious data error. Keeping them apart means you can sanity-check the research agent's output on its own — does this news item actually say what the tag claims? — before any model sees it.
**Alternative considered:** skip structured research, feed raw text straight into the prediction model. Rejected — it works until the model quietly starts keying off spurious correlations in raw text (a classic overfitting trap) instead of the actual event.

### 2. Prediction agent
**Job:** given the research agent's clean data, generate ranked signal scores per horizon (intraday / swing / long-term) — not "buy X," but "X ranks in the top decile for expected risk-adjusted return over the next week."
**Model choices:** LightGBM/XGBoost for tabular fusion of technical+fundamental+sentiment features (the workhorse); a Temporal Fusion Transformer only later, once there's enough clean historical data and a reason the simpler model isn't enough.
**Overfitting control specific to this agent:** every model here is trained with purged, embargoed walk-forward validation — never a single train/test split, never standard k-fold (both leak future information into the past). A model doesn't graduate from this agent until it beats a dumb baseline (buy-and-hold, moving average crossover) on data it never saw during training.
**Alternative considered:** deep learning from day one for everything. Rejected at this stage — deep models need more clean data than we'll have early on, and they're much easier to overfit without you noticing, because they're flexible enough to memorize noise and still look great in-sample.

### 3. Risk agent
**Job:** the gatekeeper. Takes prediction agent signals and decides position size, checks against portfolio-level limits, and can veto a signal entirely.
**Hard rules it enforces (config, not discretion):** max position size per trade, max sector/single-stock concentration, max daily loss before trading halts, max drawdown kill-switch.
**Why this is a separate agent and not a function inside prediction:** the whole point is that risk limits apply *regardless* of how confident the prediction model is. If the risk agent is just a parameter inside the prediction model, a confident-but-wrong model can talk its way past its own limits. Separation means the risk agent doesn't care how the signal was generated — it just enforces the same hard limits every time.
**Alternative considered:** Kelly criterion sizing at full strength. Rejected — full Kelly is famously aggressive and assumes your edge estimate is exactly right, which it never is early on. Fractional Kelly (typically 1/4 to 1/2) or straightforward volatility targeting is the safer default until the system has a real track record.

### 4. Execution agent
**Job:** turns an approved (post-risk-agent) signal into an actual order. **Paper trading only, until the graduation criteria in the build plan are met.**
**Why this is its own agent:** execution has its own failure modes entirely separate from prediction — slippage, partial fills, API errors, session timeouts (relevant given the Groww API's daily OAuth token expiry). Tracking these separately is how you find out whether a strategy failed because the *idea* was wrong or because the *plumbing* was wrong.

### 5. Analysis agent (the daily post-mortem)
**Job:** every evening, pulls the day's actual chart for every symbol touched, overlays the theoretical vs. actual entry/exit, computes slippage, and tags every trade's outcome with a root cause (`false breakout`, `news event`, `macro shift`, `correctly predicted but mis-sized`, `signal arrived too late`).
**Why this matters most:** this is the agent that answers your original "why did this happen" question. Without it, you have a P&L number and no idea whether the model is decaying or just had a normal bad week.
**Feedback loop:** this agent's tagged output is what future retraining and overfitting checks feed on — see the dashed arrow in the diagram. If loss attribution keeps clustering on "model correct but mis-sized," that's a risk-agent problem. If it clusters on "false breakout," that's a prediction-agent problem. The tags tell you *where* to fix, not just *that* something needs fixing.

### 6. Dashboard agent
**Job:** turns the analysis agent's output into something you can actually look at — daily/weekly P&L, win rate by strategy and horizon, drawdown curve, risk-adjusted return (Sharpe/Sortino), and the root-cause breakdown as a simple chart.
**Why separate:** reporting logic changing shouldn't risk breaking the trading logic. Also means the dashboard can run and update even on days nothing traded.

### 7. Nightly global news scan (feeds the research agent, not a standalone agent)
**Job:** overnight batch job that pulls global macro/news events (Fed decisions, geopolitical events, commodity moves, major earnings elsewhere) that could affect Indian equities the next trading day — India doesn't move in isolation from US/global sessions.
**How it works:** scheduled job → pulls from GDELT/NewsAPI/RSS → LLM pass to extract "what happened, which sectors/stocks plausibly affected, how confident" → hands structured output to the research agent before market open.
**Guardrail:** this is a *context* input, not a signal generator on its own — it flows through the same research → prediction → risk pipeline as everything else. An LLM saying "this seems bullish for IT stocks" doesn't place a trade; it becomes one more feature the prediction agent weighs alongside everything else.

---

## Where overfitting gets stopped (cross-cutting, not one agent's job)

Overfitting isn't a single agent's problem — it's a discipline enforced at every handoff:

1. **Research agent:** data is cleaned and lagged correctly (no using information that wasn't actually available at the time — a very common, very silent leakage bug).
2. **Prediction agent:** purged/embargoed walk-forward validation, always. A model that only "works" on a single backtest window is discarded, not tuned further.
3. **Model selection bias:** if we test 50 model variants and pick the best backtest, that best result is *itself* overfit to the test period by construction. The fix — hold out a final, untouched validation window that's never used for any tuning decision, only for the final go/no-go check.
4. **Risk agent:** even a genuinely good model gets capped by fixed position/drawdown limits — this bounds the damage from a model that's overfit in a way we haven't caught yet.
5. **Analysis agent:** the root-cause tagging is the early-warning system — if "model correct" tags decay over consecutive weeks, that's the signal to stop and re-validate before it decays further, not to keep trading through it.
6. **Paper trading itself is the ultimate overfitting check** — a strategy has to work on data that didn't exist when the model was built, in real time, with real friction.

---

## Open build-order question

Given "under ₹5L, variable time," building all seven agents before testing any of them would be a mistake — too much surface area to debug at once. The recommended order is: **Research → Prediction (dumb baseline model) → Risk → Execution (paper) → Analysis**, get that loop working end-to-end on one simple strategy first, *then* add the Dashboard agent and the nightly news scan, and only then start swapping in more sophisticated prediction models.

Does that build order work for you, or do you want to prioritize differently (e.g., news/sentiment first since that's the most novel piece)?
