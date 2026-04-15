# Gemini 1.5 Pro — Branch Proposal
**Date:** 2026-04-14  
**Reading from:** MASTER_BRIEF v1.0  
**Prior branches reviewed:** Claude, GPT, Grok  
**Status:** Round 1 — Initial Independent Proposal

---

## 📐 DOCUMENT METADATA

- **Document maintained by:** Gemini 1.5 Pro (Google DeepMind)
- **Last updated:** 2026-04-14
- **Next update:** After Round 2 — Challenge/Synthesis
- **Status:** Round 1 — Initial Independent Proposal

---

## 🔴 THE PRIME DIRECTIVE: PROFITABILITY ANALYSIS

Every design choice below is filtered through **Net After-Tax Profitability**. Specifically, this architecture targets "Tax Alpha" by exclusively trading Section 1256 contracts (SPX/NDX/RUT), which provides a structural ~10–15% net P&L boost compared to standard equity options due to the 60/40 tax treatment.

---

## 🏛️ PILLAR 1: DATA ARCHITECTURE & PIPELINE (SPECIAL FOCUS)

To maximize profit, we must eliminate "Information Lag." Our pipeline is built for micro-latency ingestion and high-fidelity feature engineering.

### 1.1 Ingestion Layer (The "Hydra" Pipeline)

- **Primary Source:** Tradier Streaming API (WebSockets) for real-time NBBO quotes and Greeks.
- **Secondary Source:** Databento for historical OPRA-feed level data to train models on 0DTE micro-structure.
- **Normalization:** Use FastAPI with Pydantic v2 for ultra-fast validation of incoming JSON packets.
- **Handling Data Gaps:** In the event of a Tradier stream dropout, the system auto-switches to a **"Synthetic Greeks" engine** that calculates Delta/Gamma/Theta using the underlying index price (which has higher uptime) and the last known volatility surface.

### 1.2 Feature Store (The "Chronos" Engine)

- **Technology:** QuestDB (Time-series database) for sub-millisecond queries.
- **Engineered Features:**
  - **VIX/VVIX Ratio:** Measures the "volatility of volatility" to price tail-risk.
  - **Net GEX (Gamma Exposure):** Aggregated across all SPX strikes to identify "pinning" levels and "gamma flips."
  - **Straddle-Implied Move:** Real-time calculation of the "Market Maker Move" (0.85 × Straddle Price).

---

## 🧠 PILLAR 2: REGIME DETECTION (SPECIAL FOCUS)

Profits are lost when a "Mean Reversion" strategy is applied to a "Trending" market. Our system uses a **Multi-State Hidden Markov Model (HMM)** to solve this.

### 2.1 Regime Classification

The system identifies four distinct regimes:

| Regime | Characteristics | Preferred Strategy |
|---|---|---|
| **Quiet Bullish** | Low Vol / Trending | OTM Credit Spreads |
| **Volatile Bullish** | High Vol / Expansion | Long Calls / Debit Spreads |
| **Quiet Bearish** | Low Vol / Grind Down | Bear Call Spreads |
| **Crisis / Mean Reversion** | Extreme Vol | Iron Condors or Long Puts |

### 2.2 Transition Detection & Ambiguity

- **Real-Time Detection:** We calculate the **"Viterbi Path"** every 60 seconds to determine the most likely current hidden state.
- **Regime Confidence Score (RCS):** If the HMM probability for any single regime is <60%, the RCS drops, triggering the **"Low Conviction" capital allocation tier** (30% Core / 60% Reserve).

---

## 📊 PILLAR 3: DASHBOARD & MONITORING INTELLIGENCE (SPECIAL FOCUS)

A trader's edge is only as good as their ability to interpret risk instantly.

### 3.1 The "War Room" Dashboard

- **Risk Visualization:** A real-time **Heatmap of Greek Exposure**. Instead of showing "Delta: 50," it shows a **"P&L Curve"** visualizing what happens to the portfolio if SPX moves ±1%, ±2%, or ±5% in 15 minutes.
- **Embedded AI "Sentinel":** A sidebar assistant (powered by Gemini Long-Context) that monitors the terminal and highlights anomalies.
  - _Example:_ "Alert: SPX approaching 5,200. This is a high-GEX level where 40% of today's OTM calls expire. Expect heavy resistance or a 'Gamma Squeeze'."

---

## 🔄 PILLAR 4: LEARNING ENGINE AT SCALE (SPECIAL FOCUS)

Most systems decay over time. MarketMuse is designed to grow its edge.

### 4.1 The Data Flywheel

- **Trade Tagging:** Every trade is tagged with the HMM Regime, VIX level, and GEX profile at entry.
- **Post-Mortem Analysis:** After each session, the system runs a **"Counterfactual Backtest"**: What would have happened if we held 30 minutes longer? What if we used a wider stop?
- **Self-Optimization:** On a weekly basis, the system re-calculates the **Fractional Kelly Criterion** for each strategy. If "0DTE Iron Condors" are underperforming in Regime 2, the allocation is automatically throttled until performance recovers.

---

## 🛡️ PILLAR 5: RISK MANAGEMENT (THE IRON GUARD)

- **Hard Daily Stop:** Hardcoded **-3% total portfolio drawdown limit**.
- **Dynamic Hedging:** If Delta exposure exceeds a threshold, the system automatically suggests (or executes, per V1 oversight) an offsetting XSP/SPX position to bring the portfolio back to **"Delta Neutral."**

---

## ⚡ PILLAR 6: EXECUTION ENGINE

- **Broker:** Tradier API.
- **Logic:** Use **TWAP (Time-Weighted Average Price)** for large satellite positions to minimize slippage, and **Limit Orders** for 0DTE entries to avoid the wide bid/ask spreads often seen in index options.

---

## 💡 HIGHEST-LEVERAGE INNOVATION

**The "Tax-Aware Regime Switching" Engine.** By combining the 60/40 tax edge with a regime-detection system that forces the trader into "Cash" (100% Reserve) during low-confidence periods, we maximize the "Compounding Alpha." We don't just win trades; we win the *right* trades and keep more of the profit from the IRS.

---

## ⚠️ IDENTIFIED RISKS & GAPS

| Risk | Severity | Mitigation |
|---|---|---|
| **Black Swan Events** — HMMs are historical; they may not detect a "Flash Crash" until it is already happening | Critical | Hardcoded circuit-breaker stops |
| **Slippage in RUT** — RUT options are thinner than SPX | Medium | Restrict RUT to "Satellite" status only |

---

---

# ROUND 2 — DATA & INFRASTRUCTURE AUDIT

**Date:** April 15, 2026  
**Status:** Round 2 — Challenge/Review of SYNTHESIS.md  
**Submitted by:** Gemini 1.5 Pro (Google DeepMind)

---

## DATA & INFRASTRUCTURE AUDIT — PILLAR 1: PREDICTION ENGINE

### Gap 1: The GEX "Lag-to-Loss" Loop

- **Data required:** Real-time Gamma Exposure (GEX) levels and strike-specific Open Interest (OI) updates.
- **Data reality:** Tradier (and most retail APIs) updates Open Interest once per day (pre-market). Intraday GEX changes—driven by 0DTE volume—are invisible using standard OI feeds.
- **Failure mode:** The system identifies a "Gamma Wall" at 5100 that was neutralized three hours ago. It enters a mean-reversion trade that gets run over by a trend, resulting in a max-loss stop.
- **Cold-start risk:** High. Without historical intraday OI/Volume correlations, the GEX engine is guessing on Day 1.
- **Fix:** Ingest CBOE LiveVol or Databento OPRA trade-by-trade feeds to calculate "Synthetic GEX" based on intraday volume flow rather than static morning OI.

### Gap 2: Regime Drift vs. HMM Latency

- **Data required:** 60-second Viterbi path updates from a Hidden Markov Model (HMM).
- **Data reality:** Calculating HMM states on high-dimensional feature sets (87 features) every minute is computationally expensive. If the QuestDB ingestor or the Python inference script lags by even 30 seconds, the regime signal is stale.
- **Failure mode:** The market transitions from "Quiet Bullish" to "Crisis" during a CPI print. The system, lagging by one minute, buys a "dip" that is actually the start of a liquidation event.
- **Cold-start risk:** The HMM requires a "burn-in" period. Day 1 will see frequent "state-flipping" (instability).
- **Fix:** Move HMM inference to C++ or highly optimized Rust microservices. Reduce feature set to the "Top 5" most predictive for regime detection to ensure sub-second inference.

### Gap 3: The "Broken Pipe" Signal

- **Data required:** Continuous WebSocket stream of Greeks and NBBO.
- **Data reality:** WebSockets drop. Tradier's sandbox and production environments have documented "silent timeouts" where the socket stays open but data stops flowing.
- **Failure mode:** The prediction engine "freezes" on the last known price. The market moves 0.5%, but the system thinks it's still in a "Low Vol" regime, failing to trigger an exit.
- **Cold-start risk:** Day 1 will likely suffer from unhandled connection exceptions.
- **Fix:** Implement a **Heartbeat Monitor**. If no data is received for 3 seconds, the system must immediately switch to a "Degraded Mode" (polling REST API) or flatten positions.

---

## DATA & INFRASTRUCTURE AUDIT — PILLAR 2: STRATEGY SELECTION

### Gap 1: The EV Over-Optimization Trap

- **Data required:** Real-time IV Surface to calculate Expected Value (EV) for 50+ strategy candidates.
- **Data reality:** Tradier rate limits (approx. 120 calls/min for some endpoints) make fetching the entire option chain + Greeks for SPX, NDX, and RUT simultaneously impossible during high-velocity re-scans.
- **Failure mode:** The system misses the "Optimal" strategy because the API call was throttled, settling for a sub-optimal "cached" strategy from 5 minutes ago.
- **Cold-start risk:** EV calculations will be "too perfect" (0% slippage assumed) on Day 1, leading to over-leveraging.
- **Fix:** Implement a **Priority Fetcher**. Scan only strikes within +/- 2% of Spot first. Use a local Black-Scholes/Greeks engine to calculate values locally instead of requesting them from the API.

### Gap 2: Infrastructure "Compute Debt"

- **Data required:** Monte Carlo path validation for strategy candidates.
- **Data reality:** Running 10,000 simulations per strategy for 50 candidates every 5 minutes requires a beefy multi-core server. A standard $20/mo VPS will hang.
- **Failure mode:** The "Strategy Selector" takes 4 minutes to calculate. By the time it suggests a trade, the opportunity (and the price) has moved.
- **Cold-start risk:** Total system lockup on Day 1 due to resource exhaustion.
- **Fix:** Use AWS Lambda for parallelized Monte Carlo simulations or utilize **Numba** for JIT-compiled Python performance.

### Gap 3: Tax-Alpha Ignorance

- **Data required:** Net after-tax P&L forecasting.
- **Data reality:** The synthesis assumes 60/40 tax treatment but doesn't account for "Wash Sales" or the accounting complexity of mixed-index trading.
- **Failure mode:** The system selects a strategy with higher gross profit but lower net after-tax profit because it fails to account for a disallowed loss.
- **Fix:** Integrate a **Tax-Aware Utility Function** into the EV ranking that penalizes trades with poor tax implications.

---

## DATA & INFRASTRUCTURE AUDIT — PILLAR 3: RISK MANAGEMENT

### Gap 1: The "Single Point of Failure" Dashboard

- **Data required:** Real-time account equity and drawdown metrics.
- **Data reality:** If the primary trading server goes down, the "Dashboard" (which usually runs on the same server) also dies. The trader is blind.
- **Failure mode:** A "Flash Crash" happens. The server crashes due to data overflow. The -3% Hard Stop never fires because the logic was on the crashed server.
- **Cold-start risk:** No "Emergency Manual Override" protocol tested on Day 1.
- **Fix:** Deploy an **Independent Sentinel** on a separate cloud provider (e.g., Google Cloud if the main is AWS) that only has "Close Only" permissions.

### Gap 2: Slippage Underestimation in RUT

- **Data required:** Liquidity/Spread depth for RUT options.
- **Data reality:** RUT liquidity is significantly thinner than SPX. The synthesis assumes "Institutional execution" but Tradier is a retail pipe.
- **Failure mode:** The system attempts a large Iron Condor in RUT. Slippage eats 20% of the credit on entry and 20% on exit. The "Profitable" trade becomes a net loss.
- **Fix:** Apply a **Liquidity Penalty Multiplier** to all RUT and NDX trades in the Strategy Selector.

### Gap 3: The "approval_timeout" Profit Killer

- **Data required:** Human approval within 8 minutes.
- **Data reality:** In 0DTE trading, an 8-minute window is an eternity.
- **Failure mode:** The system spots a "Gamma Squeeze" entry. The human is in the bathroom. By minute 7, the move is over. The human approves anyway (FOMO), and the trade enters at the top, immediately hitting a stop.
- **Fix:** Reduce timeout to **60 seconds**. If not approved, the trade is voided. No exceptions.

---

## THE MINIMUM VIABLE DATA STACK (ROI RANKED)

| Priority | Component | Technology | Cost/Mo | Profit Justification |
|---|---|---|---|---|
| 1 | Primary Pipe | Tradier API | $0 | Base execution |
| 2 | Time-Series | QuestDB (Self-Hosted) | $0 | Fast feature storage; essential for regime detection |
| 3 | Intraday Edge | Databento (Live Feed) | ~$150 | Highest ROI. Provides trade-by-trade data needed for real GEX |
| 4 | Off-Server Risk | AWS Lambda / SNS | ~$10 | Essential "Insurance" to kill trades if main server dies |
| 5 | Historicals | Polygon.io (1-mo sub) | $200 | One-time cost to solve the Day 1 cold-start problem |

---

## THE OPERATIONAL GAP THAT WILL CAUSE THE FIRST LIVE FAILURE

**The "WebSocket-Stale-State" Error.**

The most likely failure is that the Tradier WebSocket will drop or lag during a high-volatility event (e.g., FOMC meeting). The system will continue to calculate Greeks and EV based on "Price: 5150" when the market is actually at 5120. It will execute a "limit order" that never fills, or worse, a "market order" with 500% more slippage than expected, blowing through the daily risk limit in seconds.

---

---

# ROUND 3 — Gemini Branch Refinement (LOCKED)

**Status:** Round 3 Refinement (LOCKED)  
**Primary AI:** Gemini 1.5 Pro  
**Scope:** Q2 (Independent Sentinel) & Q4 (Databento GEX Integration)

---

## QUESTION 2 — INDEPENDENT SENTINEL FULL SPECIFICATION

The Sentinel is the "Dead Man's Switch." It must be **dumb, robust, and pessimistic.** It is deployed on **GCP (Google Cloud Platform)** to ensure a cloud-provider failure at AWS does not blind the system.

### 2.1 Tradier API Interaction

The Sentinel polls Tradier directly to verify state independently of the primary app's database.

| Endpoint | Frequency | Extracted Data | Purpose |
|---|---|---|---|
| `GET /v1/accounts/{id}/balances` | 5s | total_equity, market_value | Monitor for -3% Drawdown trigger. |
| `GET /v1/accounts/{id}/positions` | 10s | symbol, quantity, id | Maintain a list of open legs for emergency liquidation. |
| `GET /v1/markets/quotes?symbols=SPX` | 2s | last, change_percentage | Monitor for -2.5% SPX 30-min drop (Circuit Breaker). |

### 2.2 Heartbeat Mechanism: "The Pulse"

To avoid cloud-to-cloud networking complexities, the Sentinel uses a **Reverse-Proxy Health Check:**

- **Mechanism:** The Primary App (AWS) updates a dedicated heartbeat key in a TimescaleDB table (`system_health`) every 10 seconds.
- **Sentinel Logic:** The Sentinel queries this table via a read-only database replica.
- **Timeout:** If `current_timestamp - last_heartbeat > 120 seconds`, the Sentinel initiates "Degraded Check." It attempts one HTTP ping to the Primary App's health endpoint. If that fails, it assumes Primary App failure and triggers **Panic Mode (Close All).**

### 2.3 Who Monitors the Monitor?

To prevent infinite regress:

- **Sentinel Health:** The Sentinel process is wrapped in a **GCP Cloud Run "always-on" service** with a standard health check.
- **The Human Alert:** We use **UptimeRobot (External)** to monitor a public-facing health endpoint on the Sentinel. If the Sentinel stops responding, it sends a high-priority SMS/PagerDuty alert to the developer. The Sentinel is not self-healing; its failure is a "Stop Trading" event for the human.

### 2.4 Panic Mode: Sequence of Execution

When a trigger (Drawdown, SPX Drop, or Heartbeat loss) is met:

1. **Stop Primary:** Sentinel sends a `POST /v1/accounts/{id}/orders` (Cancel All Open Orders) to Tradier.
2. **Fetch Positions:** Sentinel calls `GET /v1/accounts/{id}/positions`.
3. **Order Entry:** Orders are sent in **LIFO (Last-In, First-Out)** order of entry (prioritizing 0DTE legs).
   - Sentinel uses `POST /v1/accounts/{id}/orders` with `side=sell_to_close` or `buy_to_close`.
   - **Order Type: `market`** (Profit is secondary to capital preservation during Panic).
   - **Retry Logic:** If an order fails (API error), retry every 2 seconds for 5 attempts. If it partially fills, fetch remaining quantity and re-submit as a new market order.

### 2.5 Downtime & Alerting

- **Maximum Downtime (X):** 5 Minutes.
- **Reason:** Beyond 5 minutes, the system is flying without a safety net during 0DTE volatility.
- **Alert:** If Sentinel logs to GCP Cloud Logging stop for >300s, PagerDuty triggers.

### 2.6 Sandbox Test Protocol

- **The "Tag" Method:** All paper/sandbox trades are given a `tag` field in Tradier (e.g., `MARKETMUSE_PAPER`).
- **Sentinel Filter:** During paper phase, the Sentinel is configured with a `TEST_MODE=True` flag. It will only execute `close_to_market` on orders containing the `MARKETMUSE_PAPER` tag.
- **Dry Run:** Every Friday, a "Sentinel Drill" is performed where the primary app is manually killed to ensure the Sentinel detects the heartbeat loss and simulates the closes in logs without sending the API call.

### 2.7 Sentinel Independent Logging

- **Storage:** GCP Cloud Storage (GCS) as JSON files.
- **Data:** Polled account equity, every heartbeat received, and detailed "Reason for Trigger" if Panic occurs.
- **Retention:** 90 Days (Legal/Audit requirement).

---

## QUESTION 4 — DATABENTO OPRA INTRADAY GEX INTEGRATION

Moving from static OI to intraday GEX is the "Information Alpha" that separates MarketMuse from retail competitors.

### 4.1 Databento Schema & Fields

We subscribe to the `dbn` (Binary) format for the `opranbus` (OPRA) dataset.

| Field | Required for GEX? | Purpose |
|---|---|---|
| ts_recv | Yes | Precise timing for the decay function. |
| symbol | Yes | Identify Strike, Expiration, and Type (C/P). |
| price | Yes | Calculate the spot-to-strike distance. |
| size | Yes | The quantity (the multiplier for GEX). |
| side | Yes | Critical. Used with the Lee-Ready algorithm to determine Aggressor. |
| flags | Yes | Filter out late prints or out-of-sequence trades. |

### 4.2 GEX_increment Calculation

The primary challenge is the "Aggressor Ambiguity." We use the **Lee-Ready Algorithm** to classify trades:

- **Rule:** If Trade Price > Midpoint, it is a "Buyer Aggressor" (+1). If Trade Price < Midpoint, it is a "Seller Aggressor" (-1).
- **Assumption:** A Buyer Aggressor in a Call is assumed to be an "Opening Long" (increasing Dealer Short Gamma).
- **Formula:** `GEX_increment = Aggressor_Sign * Trade_Size * Option_Gamma * 100`

### 4.3 Computational Efficiency

- **Processing:** We use the Databento Python Client with multiprocessing to handle the ~300k prints.
- **Bottleneck Fix:** We do **not** recalculate the entire IV surface for every print. We use a **Pre-computed Gamma Look-up Table (LUT)** updated every 60 seconds by the Primary App. The GEX Engine simply maps `Symbol -> LUT_Gamma` and performs a simple addition. This allows processing of 500k prints in <200ms of CPU time.

### 4.4 Fallback & Recovery

- **Detection:** If Databento `keep_alive` is missing for 10 seconds.
- **Immediate Response (0-30s):** System switches to "Static GEX Mode" (Morning Baseline).
- **Degraded Mode:** System continues to trade but applies a **50% reduction in position size** due to increased "Signal Uncertainty."
- **Confidence Decay:** GEX Confidence (C) decays linearly: `C(t) = 1.0 - (minutes_stale / 60)`. At 60 minutes of stale data, C=0 and all 0DTE entries are blocked.
- **Recovery:** Upon reconnection, the Sentinel requests a **Batch Replay** from Databento for the missed window to "catch up" the GEX_intraday total.

### 4.5 Data Join (The "Bridge")

1. **Pre-Open (08:30 AM):** Fetch CBOE DataShop EOD OI.
2. **Baseline:** Calculate GEX for every strike using the 08:30 AM Spot price.
3. **Open (09:30 AM):** Initialize `GEX_intraday = GEX_baseline`.
4. **Flow:** Start the Databento stream. Every print `p` updates `GEX_intraday += p.increment`.
5. **First Reading:** Available at 09:31 AM.

### 4.6 Validation Protocol

- **The Ground Truth:** We compare our "Synthetic OI" (Sum of Databento volume) against the next day's actual OI published by the OCC.
- **Metric:** The **"OI Tracking Error."** If our synthetic OI is within +/- 15% of the official OCC update over the 45-day paper phase, the intraday GEX engine is deemed "Production Ready."

---

*Document maintained by: Gemini 1.5 Pro (Google DeepMind) | Repo: https://github.com/tesfayekb/market-muse.git*
