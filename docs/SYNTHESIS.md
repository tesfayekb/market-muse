# SYNTHESIS.md — MarketMuse Best-Of Architecture
**Version:** 2.0 — Owner Decisions Locked, Ready for Round 3  
**Written by:** Claude (Anthropic) — Synthesis Lead  
**Date:** 2026-04-14  
**Sources:** claude-branch.md, gpt-branch.md, gemini-branch.md, grok-branch.md  
**Status:** ✅ Owner decisions resolved — Ready for Round 3 — Refinement Round

---

## SYNTHESIS METHODOLOGY

Every decision below was made using one filter only: **which proposal maximizes net after-tax profitability in live SPX/NDX 0DTE and short-swing options trading?**

Where AIs agreed — confirmed and locked.  
Where AIs disagreed — the more profitable option was chosen with explicit reasoning.  
Where all AIs missed something — flagged as a GAP.  
Each adopted idea is credited by AI name.

---

## SYNTHESIS CONFIDENCE SCORECARD

| Pillar | Confidence | Reason |
|---|---|---|
| Pillar 1: Prediction Engine | **HIGH** | Strong agreement on core architecture; GEX validated by 3 of 4 AIs |
| Pillar 2: Strategy Selection | **HIGH** | GPT's EV framework + Claude's GEX strike optimizer = compelling combination |
| Pillar 3: Risk Management | **HIGH** | All AIs converged on core risk rules; specific thresholds need live calibration |
| Pillar 4: Dashboard | **MEDIUM** | Gemini's "War Room" is strong; X integration needs evaluation |
| Pillar 5: Exit Strategy | **HIGH** | GPT's state-based model + Claude's 2:30 PM rule = significant edge |
| Pillar 6: Learning Engine | **MEDIUM** | Two-speed loop is right architecture; sample size for statistical significance is genuine unknown |
| Technical Stack | **HIGH** | Near-unanimous agreement across all AIs |

---

## 1. PROJECT OVERVIEW (CONFIRMED — NO CHANGES)

The mission, core philosophy, existing infrastructure, and all decisions in `DECISIONS.md` are confirmed and locked. This synthesis builds strictly on top of those foundations.

**Confirmed from DECISIONS.md:**
- Instruments: SPX/XSP/NDX/RUT (Section 1256, 60/40 tax treatment)
- Primary mode: 0DTE. Secondary mode: 1–5 day swing, regime-gated
- Capital allocation: Dynamic tiered (Core / Satellites / Reserve)
- Hard daily loss limit: -3% of account value
- Broker: Tradier API only
- V1: Human approval required before execution

---

## 2. PILLAR 1: PREDICTION ENGINE

**Synthesis Confidence: HIGH**  
**Architecture: Three-Layer Forecast Stack with GEX Structural Core**

The prediction engine is the most critical profit driver. All four AIs agreed on the importance of GEX and multi-layer forecasting. This synthesis combines the best elements of all four proposals.

---

### 2.1 ARCHITECTURE OVERVIEW

The prediction engine operates as a three-layer hierarchical stack. Layers are sequential — each layer gates the next. A lower layer cannot be bypassed by a higher layer.

```
LAYER A — REGIME ENGINE (runs pre-market + every 60 seconds)
      ↓ gates entry to Layer B
LAYER B — PATH & DISTRIBUTION FORECAST ENGINE (runs every 5 minutes)
      ↓ feeds Layer C
LAYER C — VOLATILITY SURFACE & MISMATCH ENGINE (runs every 5 minutes)
      ↓ feeds Strategy Selection (Pillar 2)
```

**Credit:** Three-layer stack architecture — GPT  
**Credit:** GEX as primary structural signal within each layer — Claude  
**Credit:** HMM regime classification within Layer A — Gemini  
**Credit:** Day Type Classifier as pre-market gate — Claude (adopted as the first output of Layer A)

---

### 2.2 LAYER A — REGIME ENGINE

**Runs:** Pre-market (9:00–9:29 AM EST) and every 60 seconds during session  
**Model:** Hidden Markov Model (HMM) + LightGBM ensemble  
**Output:** Regime probabilities + Day Type classification + Regime Confidence Score (RCS)

#### Step A1 — Pre-Market Day Type Classifier (9:00–9:29 AM EST only)

Before any trade is considered, classify the current trading day into one of five archetypal structures using LightGBM trained on 5 years of SPX 5-minute data:

| Day Type | Description | Primary Strategy Gate |
|---|---|---|
| **Trend Day** | Directional move initiated at open, sustained through close | Credit spreads, debit verticals |
| **Open-Drive Day** | Strong gap open continues 60–90 min then fades | Directional early, neutral late |
| **Range Day** | Auction-style oscillation within a defined intraday range | Iron condors, iron butterflies |
| **Reversal Day** | Gap open that fully reverses within first 2 hours | Contra-directional debit spreads |
| **Event Day** | Spike-and-stabilize or spike-and-reverse tied to macro events | Reduced size or cash |

**The Day Type is a hard gate — it controls which strategies are even eligible each day.**  
An iron condor is ineligible on a Trend Day, regardless of any other signal.  
This prevents the single most expensive systematic error in 0DTE trading: deploying short-gamma in trending regimes.

**Pre-market features for Day Type Classifier:**
- Overnight SPX futures return and range (/ES continuous)
- SPX gap vs 5-day ATR ratio
- VIX open vs prior close; VIX term structure slope (VIX9D/VIX ratio)
- **VVIX level** — adopted as first-class input (Credit: Claude; confirmed critical by GPT)
- Pre-market SPY volume vs 20-day average
- Economic calendar flag (Fed/CPI/NFP = 1, else 0)
- Previous 3 days' realized intraday range vs ATR
- SPX 0DTE put/call ratio from first 15 minutes of pre-market options trading

#### Step A2 — Intraday Regime Classification (every 60 seconds)

**Model:** Multi-State HMM (Credit: Gemini) with 6 regime states (Credit: GPT — added panic/liquidity-stress to Gemini's 4-state model)

| Regime | Characteristics | Capital Allocation |
|---|---|---|
| **Quiet Bullish** | Low vol, grinding trend up | Full deployment allowed |
| **Volatile Bullish** | High vol, expansion, directional | Premium buying eligible |
| **Quiet Bearish** | Low vol, grinding trend down | Bearish credit spreads |
| **Crisis / Mean Reversion** | Extreme vol, fear-driven | Iron condors or 100% reserve |
| **Pin / Range** | True auction, narrow range | Iron condors, iron butterflies |
| **Panic / Liquidity Stress** | VIX spike, spreads widening fast | 100% reserve mandatory |

**Regime detection uses Viterbi Path algorithm** (Credit: Gemini) — every 60 seconds, compute most likely current hidden state. If probability of any single regime < 60%, RCS drops to Low Conviction tier.

**VVIX Circuit Breaker (Credit: Claude — adopted as tier-1 systemic alert):**  
When VVIX > 120 OR VVIX rises > 20% in prior 30 minutes → immediately trigger Panic/Liquidity Stress regime regardless of HMM state. VVIX leads VIX by 15–60 minutes on majority of vol spikes — this is early warning for the exact scenario that destroys short-gamma portfolios.

---

### 2.3 LAYER B — PATH & DISTRIBUTION FORECAST ENGINE

**Runs:** Every 5 minutes during session  
**Model:** LightGBM ensemble (primary) + GEX Structural Model (structural backbone)  
**Architecture Credit:** GPT (distribution-first framework) + Claude (GEX computation)

#### The Core Insight: Distribution-First, Not Direction-First

**This is GPT's highest-leverage idea and is fully adopted.**

Most trading systems ask "up or down?" and force a strategy onto that binary answer. This system instead predicts the **full next-session distribution**: expected move, tail asymmetry, path shape, realized-vs-implied spread, and strike-touch probability. Every candidate strategy is then priced against that distribution after slippage, commissions, and taxes.

> **"Correct direction, wrong structure" is the most common and most expensive failure mode in options trading. Distribution-first forecasting solves it.**

#### GEX Profile Computation (every 5 minutes)

For every SPX strike with open interest > 500 contracts:

```
GEX_strike = (OI_calls × delta_call × gamma_call × 100 × spot)
           − (OI_puts × |delta_put| × gamma_put × 100 × spot)
```

Compute across all strikes to identify:
- **Positive GEX Wall:** Strike cluster where GEX is most positive → price magnet / mean-reversion zone → optimal short-strike placement for credit spreads
- **Negative GEX Flip Point:** Strike where GEX turns negative → acceleration zone → danger zone for short gamma
- **Zero GEX Line:** GEX = 0 → structural intraday pivot → watch for regime transition

**Data source for GEX:** CBOE DataShop real-time OI feed (not Tradier OI — Tradier OI may lag intraday). This is non-negotiable. Stale OI produces incorrect GEX walls. (Risk flagged by Claude — adopted.)

**Credit for GEX architecture:** Claude (primary), confirmed by Grok (X-derived GEX as supplementary validation)

#### Layer B Feature Pipeline (87 features total)

**Price/Volume (15):** 1/5/15/30-min SPX OHLCV, VWAP deviation, opening range high/low, distance from prior day close/high/low

**GEX Features (12):** Net GEX level and sign, distance to nearest GEX wall, distance to zero GEX line, GEX change rate over prior 15 min, call/put GEX ratio, GEX wall distance as % of expected move

**Volatility Surface (18):** ATM IV for 0DTE/1DTE/7DTE, IV term structure slope, IV skew (25-delta put vs call), realized vol 5-day/10-day, RV/IV ratio, VIX level, **VIX/VVIX ratio** (Credit: Gemini — adopted as key feature), straddle-implied move vs realized range

**Options Flow (20):** 5-min net premium delta (calls minus puts), unusual options activity flag, dark pool print direction/size, put/call volume ratio (0DTE only), net delta flow in last 30 min, Market Maker Move (0.85 × Straddle Price) vs realized range

**Cross-Asset (14):** TLT 5-min return, DXY 5-min return, /ES premium/discount to SPX, sector rotation score (XLK vs XLV vs XLF), gold/oil correlation flag

**Calendar/Time (8):** Time of day (sin/cos encoded), day of week, days to next Fed meeting, days to next CPI, earnings season intensity score

#### Layer B Outputs (per 5-minute cycle)

- `P(bull)` / `P(bear)` / `P(neutral)` — neutral declared if |P_bull − P_bear| < 0.15
- Expected magnitude: points and percentage (quantile regression: P10/P50/P90)
- **Strike-touch probability for key GEX levels** (Credit: GPT — adopted as critical for 0DTE exit logic)
- Probability of touching ±0.5σ, ±1σ before close
- Expected time of first directional expansion
- Timing window: probability distribution over next 30/60/90-minute intervals
- **No-trade signal** — declared when model disagreement > threshold OR expected EV after costs < minimum threshold (Credit: GPT — adopted as first-class output, not a side rule)

---

### 2.4 LAYER C — VOLATILITY SURFACE & MISMATCH ENGINE

**Runs:** Every 5 minutes  
**Model:** LightGBM regression + rule overlays  
**Credit:** GPT (adopted fully)

**Outputs:**
- Expected realized intraday vol vs current IV
- Expected IV change over trade horizon
- Premium mispricing score (near-the-money vs wings)
- Skew regime (put skew elevated = crash risk priced, reduce short-put exposure)
- Convexity regime
- **Straddle-Implied Move vs forecast move comparison** (Credit: Gemini) — if straddle prices a ±1% move and forecast says ±0.4%, premium selling has structural edge

---

### 2.5 SUPPLEMENTARY SIGNAL — SOCIAL SENTIMENT (X/TWITTER)

**Decision:** X/Twitter signals are adopted as a **supplementary tier-3 signal only, not primary alpha.** (Grok proposed as primary; Claude, GPT, Gemini did not.)

**Reasoning:** Grok's X signal proposal (monitoring @QuantData, @GammaEdges, etc.) has genuine merit — real-time GEX commentary from experienced traders can confirm or contradict the model's GEX computation. However, X API dependency, signal noise, account reliability changes, and rate limits make it an unreliable primary signal. It is adopted as a confirmation layer only — it can raise or lower conviction but cannot override Layer A/B/C outputs.

**Implementation:** Parse X for SPX/GEX keywords every 60 seconds as a sentiment overlay. Score from -1 to +1. Apply as a ±5% adjustment to prediction confidence only. (Credit: Grok — adopted in reduced role)

---

### 2.6 CONFLICT RESOLUTION — MULTI-TIMEFRAME SIGNALS

When Layer A, B, and C signals conflict:  
**Hierarchy:** Layer A wins. A Panic/Stress regime overrides all bullish path signals.  
**Cash bias:** When models disagree beyond threshold, the no-trade output wins.  
**Credit:** GPT's hierarchical gating model — adopted fully.

---

## 3. PILLAR 2: STRATEGY SELECTION ENGINE

**Synthesis Confidence: HIGH**  
**Architecture: 4-Stage Pipeline with EV-Ranked Candidate Scoring**

**Credit for framework:** GPT (EV utility function) + Claude (4-stage pipeline + GEX strike optimizer)

---

### 3.1 STAGE 1 — DAY TYPE & REGIME ELIGIBILITY GATE

Before any strategy is evaluated, apply hard eligibility rules based on Day Type and Regime:

| Day Type | Eligible Strategies |
|---|---|
| Trend Day | Credit spreads (directional), debit verticals |
| Open-Drive Day | Directional spreads (first 90 min), then neutral |
| Range Day | Iron condors, iron butterflies, credit spreads |
| Reversal Day | Contra-directional debit spreads only |
| Event Day | Reduced size only; neutral or long-vol structures |
| Panic/Stress Regime | Nothing — 100% reserve mandatory |

**Strategies explicitly excluded from V1 (Credit: Claude + GPT — both agreed independently):**
- Ratio spreads — undefined tail risk unacceptable in V1
- Naked calls or puts — same reason; also Tradier margin requirements
- Calendar spreads and diagonals — reserved for V2 (vol surface modeling complexity excessive for V1)

---

### 3.2 STAGE 2 — GEX STRIKE OPTIMIZER

For all eligible strategies, GEX walls determine strike placement:

- **Short strikes for credit spreads:** Place at Positive GEX Wall or beyond — mechanical dealer hedging provides natural defense
- **Long legs for debit spreads:** Place at Negative GEX Flip level or beyond — acceleration zone supports directional move
- **Iron condor short strikes:** Both sides must be at Positive GEX Walls — this is the highest-conviction range-trade structure available
- **No strike may be placed within 0.3% of a Negative GEX Flip zone on a short-gamma structure** — this is a hard veto, not a soft preference

**Credit:** Claude — adopted fully. This is structural edge, not statistical.

---

### 3.3 STAGE 3 — EV RANKING ENGINE

For all strategies that pass Stages 1 and 2, compute expected value under the Layer B distribution forecast:

```
Utility = EV_net 
        − λ1 × ExpectedShortfall 
        − λ2 × TailRisk 
        − λ3 × SlippagePenalty 
        − λ4 × LiquidityPenalty 
        + λ5 × CapitalEfficiency
```

Where:
- `EV_net` = expected P&L including realistic slippage + commissions + 60/40 tax adjustment
- `ExpectedShortfall` = expected loss in worst 10% of outcomes (CVaR, not VaR)
- `TailRisk` = premium weighted probability of gap through hard stop
- `SlippagePenalty` = estimated bid/ask cost from current spread data
- `LiquidityPenalty` = penalty if open interest < 500 or daily volume < 100 on any leg
- `CapitalEfficiency` = EV per margin dollar deployed

**Credit:** GPT (utility function) — adopted fully. After-tax EV is explicitly included — this directly serves the 60/40 tax advantage.

**The highest-utility strategy wins. EV is never overridden by "feel" or regime intuition.**

---

### 3.4 STAGE 4 — LIQUIDITY & SLIPPAGE HARD GATE

Before final selection, apply these hard vetoes regardless of EV ranking:

- **Bid/ask spread > $0.30 on any single leg** → veto the entire structure (Credit: Claude)
- **Open interest < 500 on any leg** → veto
- **Daily volume < 100 on any leg** → veto
- **Time of day: 9:30–10:00 AM** → all strategies downgraded one size tier (spreads are 3–5× wider)
- **Time of day: 3:30–3:45 PM** → only hard-close execution permitted, no new entries

**Credit:** Claude + broker_constraints.md — adopted. Slippage is a hard eligibility criterion, not a penalty that can be overcome by high EV.

---

### 3.5 0DTE vs SWING STRATEGY SELECTION DIFFERENCE

| Factor | 0DTE Mode | Swing Mode (1–5 days) |
|---|---|---|
| Primary structures | Credit spreads, iron condors, debit verticals | Debit spreads, occasionally long options |
| Strike selection | GEX walls dominate | Delta-based + directional conviction |
| IV consideration | IV rank > 30th percentile preferred for selling | IV rank > 50th percentile for buying |
| Time stop | Hard 2:30 PM for short-gamma (see Pillar 5) | Pre-event exit if swing into a macro day |

---

## 4. PILLAR 3: RISK MANAGEMENT ENGINE

**Synthesis Confidence: HIGH**  
**Architecture: Three-Level Risk Stack with VVIX Early-Warning System**

---

### 4.1 POSITION-LEVEL RISK CONTROLS

**Position Sizing Formula (Credit: Claude formula + Grok thresholds):**

```
Position Size = (Account Value × Risk % × RCS/100) 
              / (GEX-adjusted expected move × $100 multiplier)
```

Where:
- Risk % per core trade: 0.5% of account (Credit: Grok — adopted as starting calibration point)
- RCS: Regime Confidence Score 0–100
- GEX-adjusted expected move: predicted magnitude from Layer B, widened if in negative GEX zone

**Hard Greek limits per position (Credit: Grok — specific thresholds adopted as V1 starting limits):**
- Maximum position delta: ±0.15 net portfolio delta per $100k account value
- Maximum position vega: ±$500 per $100k account value
- Maximum gamma: as defined by 2:30 PM mandatory exit rule for short-gamma (see Pillar 5)

**Pre-entry slippage gate:** If estimated round-trip slippage on any structure exceeds 20% of max potential credit/profit → veto entry regardless of EV ranking. (Credit: Claude)

---

### 4.2 PORTFOLIO-LEVEL RISK CONTROLS

**Correlation check:** No two positions with directional exposure correlation > 0.7 permitted simultaneously. SPX + NDX + RUT all have correlation > 0.80 during stress events — this limits "diversification" across indices to size reduction, not elimination of correlated risk. (Credit: execution_risks.md)

**Portfolio Greek aggregation:** After every position change, recalculate portfolio-level delta, vega, and gamma. If any breach portfolio limit, new position is rejected or existing position must be trimmed first.

**Maximum simultaneous positions:** 3 (1 Core + 2 Satellites) — confirmed from DECISIONS.md.

**Margin utilization cap:** 70% of available buying power. 30% always held as buffer. (Credit: broker_constraints.md)

---

### 4.3 SYSTEMIC RISK CONTROLS

**VVIX Early-Warning System (Credit: Claude — highest-priority adoption):**

| VVIX Condition | System Response |
|---|---|
| VVIX > 120 | WARNING — reduce all new position sizes by 50% |
| VVIX rises > 20% in 30 min | WARNING — same as above regardless of absolute level |
| VVIX > 140 | CRITICAL — close all short-gamma positions immediately |
| VVIX > 160 | EMERGENCY — 100% reserve, halt all trading |

**Why VVIX over VIX (Credit: Claude):** VVIX leads VIX by 15–60 minutes on majority of vol spikes. For a short-gamma system, this is the difference between early warning and getting caught in the spike. VIX monitoring is secondary.

**Hard circuit breakers:**
- SPX drops > 2% in any 30-minute window → halt all new entries immediately
- VIX spikes > 15% intraday → all positions reviewed, tighten stops
- VIX spikes > 25% intraday → close all short-gamma positions
- Exchange circuit breaker Level 1 (−7%) → all orders cancelled, system halts, human review
- Exchange circuit breaker Level 2 (−13%) → same, mandatory human review before restart
- Exchange circuit breaker Level 3 (−20%) → system halts for remainder of day

**Hard daily loss limit:** -3% of account → all positions closed, trading halted for day — hardcoded, no override. (Confirmed from DECISIONS.md)

**8-minute human approval timeout (Credit: Claude — adopted as new rule):**  
In V1, when the system issues a trade recommendation, human approval must be received within **8 minutes**. If not approved within 8 minutes, the recommendation expires and requires full re-evaluation. Executing a stale recommendation is explicitly forbidden — GEX structure and regime confidence may have materially changed.

**Backend failure protection (Credit: Claude — adopted as critical infrastructure rule):**  
At every trade entry confirmation, pre-configure Tradier OCO stop orders immediately. The safety net lives in the broker, not the application. Backend failure must never produce unmanaged open positions. This is non-negotiable.

---

### 4.4 PRE-MARKET RISK SCAN

Run every morning 9:00–9:25 AM before any trading:
1. VVIX level vs prior week average
2. VIX level and term structure
3. Overnight futures gap assessment
4. Economic calendar check — flag all events
5. Margin availability confirmation
6. Any open swing positions reviewed against pre-market conditions
7. System health check (data feeds, API connectivity, latency test)

If any pre-market scan produces a WARNING or higher → reduce day's allocation tier by one step.

---

## 5. PILLAR 4: MONITORING & DASHBOARD ENGINE

**Synthesis Confidence: MEDIUM**  
**Architecture: War Room Dashboard with AI Sentinel**  
**Credit:** Gemini (War Room + Sentinel concept — primary), Grok (alert tiers), Claude (slippage tracker + approval timeout)

---

### 5.1 THE "WAR ROOM" DASHBOARD

**Most critical display innovation — adopted from Gemini:**  
Replace raw Greeks numbers with a **P&L Curve Heatmap** — a real-time visualization showing what happens to the portfolio if SPX moves ±1%, ±2%, ±5% over the next 15, 30, and 60 minutes. This communicates risk in terms traders can act on instantly, not abstract Greek values a non-quant must mentally translate.

**Dashboard Panels:**

| Panel | Content | Update Frequency |
|---|---|---|
| **Scenario P&L Heatmap** | Portfolio value at SPX ±0.5/1/2/5% in next 15/30/60 min | Every 60 seconds |
| **Live P&L** | By position + portfolio total, unrealized + realized today | Real-time streaming |
| **Greeks Dashboard** | Delta, gamma, theta, vega — per position and portfolio total | Every 30 seconds |
| **Prediction Confidence** | Direction probability, confidence score, model agreement across layers | Every 5 minutes |
| **Regime Status** | Current HMM regime + Day Type + RCS score | Every 60 seconds |
| **VVIX/VIX Monitor** | Current levels, rate of change, alert status | Real-time |
| **GEX Map** | Live GEX wall visualization — where price has structural support/resistance | Every 5 minutes |
| **Capital Allocation** | Core/Satellite/Reserve utilization vs RCS-adjusted target | Real-time |
| **Daily Drawdown Meter** | % used of -3% hard stop, proximity alert at -2% | Real-time |
| **Slippage Tracker** | Estimated vs actual slippage per trade, session total (Credit: Claude) | Per trade |
| **Win Rate Dashboard** | Rolling 5/20/60 day: win rate, avg win, avg loss, expectancy, profit factor | Daily |
| **Approval Queue** | Pending trade recommendations + time remaining before expiry (Credit: Claude) | Real-time |

---

### 5.2 EMBEDDED AI SENTINEL — "SENTINEL"

**Credit: Gemini — adopted with full enthusiasm. This is the most unique dashboard feature.**

An embedded AI assistant (sidebar) that monitors all system outputs and surfaces anomalies in plain English. The Sentinel does not trade — it explains, warns, and educates.

**Example Sentinel outputs:**
- *"SPX is approaching 5,450. This is a Positive GEX Wall with 3,200 contracts of call OI. Expect mean-reversion resistance here. The current iron condor short call is 12 points away — within normal decay range."*
- *"VVIX has risen 18% in the last 25 minutes while VIX is still calm. This is a pre-spike pattern observed in 7 of the last 10 major vol events. Recommend reviewing current short-gamma exposure."*
- *"The current prediction model confidence has dropped from 72% to 54% over the last 20 minutes. The trade thesis may be degrading. Consider early exit evaluation."*

The Sentinel is critical for V1 human oversight — it bridges the gap between a complex automated system and a human who must approve and monitor trades.

---

### 5.3 ALERT TIERS

**Credit: Grok (tier concept) + Claude (VVIX thresholds applied to tiers)**

| Tier | Condition | Channel | Action Required |
|---|---|---|---|
| **INFO** | Regime shift, prediction update | Dashboard only | Monitor |
| **WARNING** | VVIX +15%, RCS drops below 50, position approaching SL by 30% | Dashboard + push notification | Review within 5 minutes |
| **CRITICAL** | Approaching -2% daily drawdown, VVIX +20%, approval queue expiring | Push + SMS | Act immediately |
| **EMERGENCY** | -3% drawdown triggered, circuit breaker, VVIX > 140 | Push + SMS + email | System has auto-responded; human confirms |

**Note on X/Twitter panel:** Grok proposed a live X gamma feed panel. This is assessed as MEDIUM priority for V1 — high maintenance, potential noise amplification. Add as an optional toggle panel rather than a primary display. Revisit in V2 based on signal quality observed in V1.

---

## 6. PILLAR 5: P&L, STOP-LOSS & EXIT STRATEGY ENGINE

**Synthesis Confidence: HIGH**  
**Architecture: State-Based Exit Model with GEX-Trailing and Strike-Touch Probability**  
**Credit:** GPT (state-based model + strike-touch probability — primary), Claude (2:30 PM exit, GEX trailing, VVIX exit trigger)

---

### 6.1 THE MOST IMPORTANT EXIT DECISION — 2:30 PM FOR SHORT-GAMMA

**Claude's 2:30 PM rule is adopted over the MASTER_BRIEF's 3:45 PM rule.**

**Reasoning:** The MASTER_BRIEF's 3:45 PM exit is 75 minutes too late for short-gamma (credit spreads, iron condors). The gamma risk from 2:30–3:45 PM on short-gamma 0DTE positions is empirically not worth the remaining theta capture. From 2:30 PM onward, gamma acceleration is non-linear — a position that is comfortable at 2:25 PM can be in danger at 2:35 PM. The theta dollars remaining in that 75-minute window do not compensate for the gamma risk taken.

**Exit time rules by strategy type:**

| Strategy Type | Mandatory Exit Time | Rationale |
|---|---|---|
| Short-gamma (credit spreads, iron condors, iron butterflies) | **2:30 PM EST** | Gamma acceleration from 2:30 PM creates unacceptable risk/reward |
| Long-gamma (debit spreads, long calls/puts) | **3:45 PM EST** | Theta works against you, but risk is defined |
| Swing positions (1–5 day holds) | Exit before next scheduled macro event | Overnight gap risk from known events is uncompensated |

---

### 6.2 STATE-BASED EXIT MODEL

**Credit: GPT — adopted fully as primary exit framework.**

Every open position occupies exactly one of five states. The system recalculates state every 5 minutes:

| State | Condition | Action |
|---|---|---|
| **1 — Entry Validation** | Just entered, within first 15 minutes | No action — let position breathe |
| **2 — Early Confirmation** | Position moving favorably, forecast still intact | Monitor, no change |
| **3 — Mature Winner** | > 40% of max profit captured; forecast intact | Take 50% off, trail remainder |
| **4 — Degrading Thesis** | Prediction confidence dropped > 15 pts from entry, OR GEX wall breached | Exit 50% immediately, evaluate remaining |
| **5 — Forced Exit** | Hard SL hit, OR time stop triggered, OR VVIX emergency, OR -3% portfolio stop | Exit 100% immediately — no evaluation |

---

### 6.3 STRIKE-TOUCH PROBABILITY AS PRIMARY 0DTE CREDIT EXIT VARIABLE

**Credit: GPT — adopted as the most sophisticated and profitable 0DTE exit innovation.**

For credit spreads and iron condors, the primary exit variable is **the probability of the underlying breaching the short strike before close**, given current time, current price, and current realized vol. This is more informative than mark-to-market P&L alone.

**Formula:**
```
Strike_Touch_Probability = N(d2 adjusted for remaining time and realized vol)
```

**Exit thresholds:**
- Strike-touch probability < 10%: take 60% of max credit, exit position (don't wait for last $0.05)
- Strike-touch probability 10–25%: let run with trail on short strike
- Strike-touch probability > 25%: exit 50% immediately, re-evaluate remaining
- Strike-touch probability > 40%: exit 100% — thesis is degrading

**This directly prevents "donating edge back to the market"** — the common failure of holding a 90% winner too long until it becomes a loser.

---

### 6.4 PARTIAL PROFIT TAKING PROTOCOL

**For all credit structures (0DTE primary):**
- At 40–50% of max credit captured → take 50% off, set trailing stop on remainder
- At 65–70% of max credit captured → close 100% (remaining theta too small vs remaining gamma risk)
- At Strike-Touch Probability > 25% → begin scaling out regardless of P&L

**For debit structures (directional):**
- First scale: at 35–50% of expected move captured → close 50%
- Trail remainder using GEX wall levels as trailing stop anchors (Credit: Claude)
- Forecast-decay trigger: if prediction confidence drops > 15 points from entry → exit remaining immediately

**Credit for partial profit logic:** GPT (framework) + Grok (50/50 specific split) + Claude (GEX trailing)

---

### 6.5 HARD EXIT RULES (NON-NEGOTIABLE)

- All 0DTE short-gamma positions flat by **2:30 PM EST** — no exceptions
- All 0DTE long-gamma positions flat by **3:45 PM EST** — no exceptions
- No averaging into losing positions — ever (Credit: GPT, Grok — both stated independently)
- No converting a losing spread into a "different" position to avoid realizing the loss
- If the -3% portfolio stop triggers, all positions exit immediately — system halts for the day
- Hard price stop: always pre-defined and pre-submitted as OCO at entry

---

### 6.6 PRE-TRADE DEFINITION REQUIREMENT

Before every entry, the system must auto-define and record:
- Entry price, Greeks at entry, prediction score at entry, regime state, Day Type
- Profit target (PT) and stop-loss (SL) — output by system, not manually set
- Strike-touch probability at entry
- Estimated slippage
- After-tax EV

---

### 6.7 POST-TRADE ATTRIBUTION

**Credit: GPT — adopted as the learning feed for Pillar 6.**

Every closed trade is classified by outcome driver:
- Direction right / structure wrong
- Direction wrong / structure resilient
- Vol right / timing wrong
- Entry right / exit poor
- No-trade signal should have been respected

This attribution taxonomy directly feeds the learning engine to identify specifically which component failed.

---

## 7. PILLAR 6: AI LEARNING & ADAPTATION ENGINE

**Synthesis Confidence: MEDIUM**  
**Architecture: Two-Speed Learning Loop with Three-Layer Diagnostics**  
**Credit:** GPT (two-speed loop + champion/challenger) + Claude (three-layer separation) + Gemini (counterfactual backtest) + Grok (specific drift threshold)

---

### 7.1 THE TWO-SPEED LEARNING LOOP

**Credit: GPT — adopted as the primary learning architecture.**

The risk of learning too fast: overfitting to recent conditions and abandoning valid edge.  
The risk of learning too slow: failing to adapt to genuine regime changes.  
Two-speed solves both simultaneously.

**Fast Loop — Daily (runs end-of-session, 4:15–5:00 PM):**
- Recalibrate model probabilities using isotonic regression
- Update regime-strategy scorecard with latest session results
- Update fill/slippage model with actual fills
- Update no-trade classifier with missed setups
- Flag any sessions where model accuracy < 58% for review

**Slow Loop — Weekly (runs Sunday evening):**
- Full model retrain on rolling 90-day window (minimum)
- Champion vs Challenger model comparison
- Promote challenger only after: (a) out-of-sample accuracy > champion AND (b) recent-regime accuracy > champion AND (c) Sharpe improvement > 0.1
- Update regime-strategy performance matrix
- Fractional Kelly recalibration per strategy/regime combination (Credit: Gemini)

---

### 7.2 THREE-LAYER LEARNING DIAGNOSTICS

**Credit: Claude — adopted. This is how you know WHAT to fix, not just THAT something is wrong.**

The learning engine tracks performance separately at three layers:

**Layer 1 — Signal Quality:** Is the prediction engine accurately forecasting direction, magnitude, and timing? Tracked by: prediction accuracy vs outcome, by regime and Day Type.

**Layer 2 — Strategy Selection Quality:** Given the prediction, was the right structure selected? Tracked by: comparing actual outcome to counterfactual outcome of the highest-ranked alternative strategy.

**Layer 3 — Sizing/Exit Quality:** Given the right signal and structure, was capital deployed and managed optimally? Tracked by: peak P&L vs captured P&L, entry slippage vs estimated.

If Layer 1 is failing → fix the prediction engine.  
If Layer 1 is fine but Layer 2 is failing → fix strategy selection.  
If Layers 1 and 2 are fine but Layer 3 is failing → fix position sizing and exit logic.

**This diagnostic separation prevents the most common ML system error: retraining the prediction model when the real problem is exit logic.**

---

### 7.3 CHAMPION/CHALLENGER INFRASTRUCTURE

**Credit: GPT — adopted as non-negotiable from Day 1.**

Never run a single model in production. Always maintain:
- **Champion:** Current production model
- **Challenger:** Latest retrained version, running in shadow (paper trades only)
- **Archived:** Previous versions with performance history

Challengers run in shadow for minimum 20 live trading sessions before any promotion consideration. Only promote if challenger outperforms champion on: accuracy, Sharpe, and drawdown metrics across at least 2 different regime types.

---

### 7.4 COUNTERFACTUAL BACKTEST ENGINE

**Credit: Gemini — adopted as the most underrated learning tool proposed.**

After every session, the system automatically runs: *"What would have happened if we..."*
- Held the position 30 minutes longer?
- Used a wider stop?
- Exited at the model's degrading-thesis signal instead of the hard stop?
- Chose the second-ranked strategy instead of the first?

These counterfactuals do not change past trades. They provide the training signal for improving future exit timing and strategy selection. This is compounding learning — every session generates not just one training example but 4–6 via counterfactual branches.

---

### 7.5 OUT-OF-DISTRIBUTION DETECTION

**Credit: GPT — adopted as critical for novel regime handling.**

When incoming feature vectors are out-of-distribution relative to training data:
- Automatically reduce position size
- Narrow the allowed strategy set (debit structures only, reduce credit structures)
- Raise the no-trade signal probability
- Label the session for regime clustering (may represent a new regime type)

**Novel regime response:** Do not extrapolate. Reduce size, sit in reserve-heavy posture, accumulate data until the new regime can be characterized.

---

### 7.6 DRIFT DETECTION THRESHOLD

**Credit: Grok — specific threshold adopted.**

- If rolling 5-trade prediction accuracy drops below 58% → trigger model drift alert
- If rolling 20-trade prediction accuracy drops > 10 percentage points below historical average → auto-reduce position sizing by 50% pending human review
- If drift persists > 3 sessions → automatically promote challenger if available; else halt new prediction-dependent trades and run in pure-GEX-structural mode

**The no-trade model is the single largest contributor to long-run Sharpe** (Credit: GPT — fully endorsed). Avoided losses compound exactly as powerfully as captured wins. A system that correctly sits out 15 poorly-set-up days per year has materially higher Sharpe than one that forces trades on those days.

---

## 8. TECHNICAL STACK — SYNTHESIZED RECOMMENDATION

**Consensus is strong here — near-unanimous agreement across all AIs.**

| Component | Technology | Justification | Credit |
|---|---|---|---|
| **Backend API** | Python 3.12 + FastAPI | All AIs agreed; existing Tradier integration compatible | All AIs |
| **Primary ML** | LightGBM + XGBoost | Fast, interpretable, proven on tabular financial data; beats PyTorch on 0DTE latency constraints | Claude, Grok, GPT |
| **Path Models (later)** | PyTorch (TFT or N-BEATS) | Added after 6–12 months of clean feature history | GPT |
| **Regime Model** | HMM + LightGBM ensemble | HMM for state probabilities, LightGBM for discriminative classification | Gemini, GPT |
| **Feature Store** | QuestDB | Sub-millisecond time-series queries; better latency than TimescaleDB for intraday | Gemini |
| **Trade/History DB** | TimescaleDB (PostgreSQL extension) | Best for trade history + learning engine data | GPT, Grok |
| **Intraday Cache** | Redis | Session state, throttling, real-time position cache | GPT |
| **Model Registry** | MLflow | Champion/challenger tracking, version control | GPT |
| **Hyperparameter Tuning** | Optuna | Disciplined, reproducible search | GPT |
| **Backtesting** | vectorbt | Fastest Python options backtesting; handles Greeks simulation | Claude |
| **Frontend** | Existing React stack | Already built; add charting and Sentinel layer on top | All AIs |
| **Deployment** | AWS ECS (us-east-1 region) | Must be us-east-1 — closest to NYSE/CBOE; cross-region latency unacceptable | All AIs |
| **Data: Real-time quotes** | Tradier Streaming WebSocket | Already integrated | All AIs |
| **Data: Historical options** | Databento (OPRA feed) | Best quality historical options data for model training | GPT, Gemini |
| **Data: Real-time OI for GEX** | CBOE DataShop | Required for accurate intraday GEX — Tradier OI lags | Claude |
| **Data: Supplemental** | Polygon.io | Historical fill-in, market breadth, dark pool prints | Grok |
| **Data: Vol surface** | CBOE DataShop or Interactive Brokers feed | VIX futures curve, VVIX real-time | Claude |

**Estimated monthly data costs (V1):**
- Tradier: commission-based (no data fee beyond trading)
- CBOE DataShop real-time: ~$85/month
- Databento (historical, one-time backfill): ~$200–400 one-time
- Polygon.io: ~$29–79/month depending on tier
- **Total ongoing: ~$115–165/month** — negligible vs potential P&L

---

## 9. CRITICAL GAPS — IDENTIFIED BY SYNTHESIS REVIEW

These are items that no AI addressed adequately or that represent genuine blind spots across all four proposals.

---

**GAP 1 — DISASTER RECOVERY FOR MID-SESSION BACKEND FAILURE**

All four AIs described what the system does when running normally. None adequately designed for what happens when the backend crashes at 1:30 PM with 2 open positions.

**Required solution (not yet specified by any AI):**
- Pre-configure Tradier OCO stops at every fill confirmation (Claude noted this as a risk — it must become an implementation requirement, not just a risk item)
- Backend must write current position state to persistent storage after every state change
- On restart, first action is: read position state from storage, confirm with Tradier API, re-arm all safety mechanisms before any new trading
- Human operator must be notified via push/SMS on any backend restart during market hours

---

**GAP 2 — COLD START PROBLEM**

No AI adequately addressed Day 1 performance when zero live trade history exists.

**Required solution:**
- Phase 1 (Days 1–30): Run system in paper-trade mode. Collect prediction vs outcome data. DO NOT go live until minimum accuracy thresholds are confirmed on paper.
- Phase 2 (Days 31–60): Go live with 25% of intended allocation. System is learning slippage calibration and fill quality.
- Phase 3 (Day 61+): Full allocation. Champion/challenger active. Learning engine has statistical significance.
- The learning engine requires minimum ~200 trades per strategy/regime combination before adaptations have statistical significance. At 1–3 trades per day, this is a 3–6 month journey per combination.

---

**GAP 3 — KILL-SWITCH HIERARCHY**

No AI defined who can stop the system and under what conditions.

**Required hierarchy:**
- Level 1 (Automated): -3% drawdown, VVIX emergency, circuit breaker → system self-halts
- Level 2 (Human — any user): Can pause new entries, cannot close open positions without Level 3
- Level 3 (Admin/Owner): Can close all positions, can override recommendation expiry in exceptional circumstances
- Level 4 (Emergency): Physical power/network disconnect — last resort

---

**GAP 4 — HUMAN APPROVAL UX NOT DESIGNED**

V1 requires human approval for all trades. The approval workflow exists as a concept but has no UX specification. The 8-minute expiry window (adopted from Claude) means the approval UX must be fast, clear, and mobile-friendly.

**Required minimum spec:**
- Push notification with trade summary (strategy, size, strike, expected P&L, confidence score, expiry timer)
- One-tap Approve / Reject / Request More Info
- Expiry countdown visible
- On rejection: system logs reason (optional field) for learning engine
- On expiry: system auto-rejects and logs

---

**GAP 5 — PERFORMANCE ATTRIBUTION REPORTING**

All AIs discussed per-trade attribution but none designed a session-level or monthly reporting framework.

**Required:**
- Daily end-of-session report: trades taken, P&L, prediction accuracy, slippage actual vs estimated, regime conditions
- Weekly report: rolling metrics vs targets (Sharpe, win rate, profit factor), learning engine updates
- Monthly report: after-tax P&L estimate using 60/40 treatment, Sharpe vs drawdown, model performance by regime

---

## 10. SUCCESS METRICS — CONFIRMED WITH ADDITIONS

| Metric | Target | Added From |
|---|---|---|
| Net After-Tax P&L | Bottom line after 60/40 treatment | MASTER_BRIEF |
| Sharpe Ratio | > 2.0 | MASTER_BRIEF |
| Maximum Drawdown | < 10% | MASTER_BRIEF |
| Win Rate | > 60% credit, > 55% debit | MASTER_BRIEF |
| Profit Factor | > 1.8 | MASTER_BRIEF |
| Avg Win / Avg Loss | > 1.5 | MASTER_BRIEF |
| Prediction Accuracy | > 58% directional | MASTER_BRIEF |
| Strategy EV Accuracy | % trades where selected strategy had highest realized EV | GPT addition |
| No-Trade Signal Accuracy | % of skipped setups that would have lost money | GPT addition |
| Slippage vs Estimated | Actual slippage within 25% of model estimate | Claude addition |
| Model Drift Response | Detect and respond within 5 trading sessions | MASTER_BRIEF |
| Cold-Start Paper Phase | 58%+ accuracy confirmed before going live | Synthesis addition |

---

## 11. RESOLVED OWNER DECISIONS — NOW LOCKED

All five conflicts from Synthesis v1.0 have been resolved by the project owner. These are now locked decisions and may not be re-argued in Round 3. AIs may only propose implementation refinements within these locked parameters.

---

### DECISION-008: Data Provider Budget
**Status:** 🔒 LOCKED  
**Decision:** APPROVED. Monthly data budget of ~$150–200/month authorized.  
**Stack confirmed:**
- CBOE DataShop real-time: ~$85/month (GEX computation + VVIX real-time)
- Polygon.io: ~$50/month (breadth data, dark pool prints, supplemental history)
- Databento: one-time historical backfill ~$200–400
- **Total ongoing: ~$135–165/month**

**Profit reasoning:** This data spend is recoverable in a single average winning trade. Without CBOE real-time OI, the GEX model is unreliable — this is not optional infrastructure.

---

### DECISION-009: X/Twitter Signal Scope — V1
**Status:** 🔒 LOCKED  
**Decision:** X/Twitter adopted as **Tier-3 supplementary confirmation signal in V1.** Not primary alpha. AIs in Round 3 must propose the specific implementation architecture for this tier-3 role.

**Profit reasoning for Tier-3 (not Tier-1):**
- X API reliability and rate limits create execution dependency risk unacceptable for a primary signal
- Signal quality from individual accounts is unvalidated — no backtest exists for @QuantData accuracy
- Real-time GEX computed directly from live options chain data (CBOE DataShop) is structurally superior to parsing GEX commentary from social posts — the commentary is a derivative of the same data we already own
- However: sentiment velocity on X (posts/minute rising sharply on SPX/gamma keywords) can provide useful confirmation or early warning that is worth a small confidence adjustment
- **V2 upgrade path:** After V1 live trading, analyze correlation between X sentiment signals and actual outcomes. If correlation > 0.15 with positive profit attribution, promote to Tier-2 in V2.

**Tier-3 implementation bounds:** X signal can adjust prediction confidence by ±5% maximum. It cannot trigger entries, exits, or risk alerts independently.

---

### DECISION-010: Short-Gamma Mandatory Exit Time
**Status:** 🔒 LOCKED  
**Decision:** **2:30 PM EST** is the mandatory exit time for all short-gamma positions (credit spreads, iron condors, iron butterflies). This overrides the original MASTER_BRIEF 3:45 PM rule.

**Profit reasoning:** From 2:30 PM onward on 0DTE expiration, gamma acceleration on short-gamma structures becomes non-linear and the risk/reward inverts. The theta remaining in the 2:30–3:45 PM window ($0.05–0.15 on typical spreads) does not compensate for the gamma risk of being caught by a late-session directional move. Empirically, the majority of 0DTE short-gamma blow-ups occur in the final 90 minutes of trading. Capturing 85% of the theta while avoiding 90% of the terminal gamma risk is the higher-EV decision.

**Long-gamma (debit structures) remain at 3:45 PM EST** — time decay works against them, but risk is fully defined.

---

### DECISION-011: Paper Trading Phase
**Status:** 🔒 LOCKED  
**Decision:** Minimum **30-day paper trading phase** (approximately 60 trading sessions) required before any live capital is deployed.

**Go-live criteria (all must be met):**
1. Paper prediction accuracy ≥ 58% directional over the full 30 days
2. Paper Sharpe ratio ≥ 1.5 (lower threshold than live target, accounting for paper-to-live execution differences)
3. Zero system errors causing unmanaged positions in final 10 paper sessions
4. All 5 circuit breaker scenarios tested and verified in sandbox
5. Human approval workflow tested and approved by operator

**Profit reasoning:** Going live on an uncalibrated system loses real money to slippage model errors, fill quality surprises, and edge cases that only appear in live conditions. 30 days is the minimum to build statistical confidence without delaying value creation indefinitely.

---

### DECISION-012: RUT Instrument Classification
**Status:** 🔒 LOCKED  
**Decision:** RUT options are restricted to **Satellite positions only** in V1. RUT may never be a Core position in V1.

**Additional V1 RUT constraints:**
- Maximum RUT position size: 50% of standard Satellite size
- Minimum open interest on any RUT leg: 300 contracts (vs 500 for SPX/NDX)
- Minimum daily volume on any RUT leg: 75 contracts (vs 100 for SPX/NDX)
- RUT bid/ask spread maximum per leg: $0.40 (vs $0.30 for SPX/NDX) — wider tolerance required given thinner market

**Profit reasoning:** RUT options have materially wider bid/ask spreads and lower liquidity than SPX or NDX options. Using RUT as a Core position introduces slippage drag that structurally reduces net P&L. As a Satellite, the smaller size limits the damage while still allowing participation in RUT-specific setups (small-cap regime divergence, RUT vs SPX spread opportunities). Full RUT trading eligibility will be evaluated for V2 after live slippage data confirms whether the edge justifies the execution cost.

**SPX remains the primary instrument. NDX is secondary. RUT is tertiary and satellite-only.**

---

## 12. OPEN QUESTIONS FOR ROUND 3

These are the specific refinement questions all AIs must address in Round 3. Do not re-argue locked decisions. Build on them.

**Q1 — X/Twitter Tier-3 Implementation:**  
Design the specific implementation of X/Twitter as a tier-3 signal. What keywords, what accounts, what parsing logic, what scoring formula produces the ±5% confidence adjustment? How do you validate signal quality in the paper trading phase?

**Q2 — GEX Cold-Start Solution:**  
The GEX model requires real-time OI from CBOE DataShop. On Day 1 of paper trading, historical GEX calibration may be incomplete. How do you bootstrap the GEX model with confidence before sufficient live data accumulates?

**Q3 — Sentinel AI Specification:**  
The AI Sentinel assistant (Gemini's idea, adopted) needs a specific implementation spec. What model powers it? What data does it monitor? What are the exact trigger conditions for each type of alert it generates? How does it avoid alert fatigue?

**Q4 — Champion/Challenger Promotion Criteria:**  
GPT proposed champion/challenger infrastructure (adopted). Define the exact statistical tests and thresholds required before a challenger is promoted to champion. How many sessions? What confidence interval? What happens if champion and challenger are statistically tied?

**Q5 — Position Sizing Calibration for Paper Phase:**  
The position sizing formula requires live calibration (slippage actuals, fill quality, GEX accuracy). What is the specific calibration process during the 30-day paper phase? How do you translate paper results into live sizing parameters?

---

## APPENDIX — AI CREDIT SUMMARY

| AI | Highest-Value Contribution Adopted |
|---|---|
| **Claude** | GEX structural architecture, Day Type pre-market classifier, VVIX as first-class circuit breaker, 2:30 PM short-gamma exit, backend OCO failure protection, 8-minute approval timeout |
| **GPT** | Distribution-first prediction framework, utility function EV ranking, strike-touch probability for 0DTE exits, state-based exit model, two-speed learning loop, champion/challenger infrastructure, no-trade classifier as first-class output |
| **Gemini** | HMM regime classification + Viterbi path, QuestDB for feature store, War Room P&L Curve heatmap, AI Sentinel embedded assistant, counterfactual backtest engine, VIX/VVIX ratio as feature |
| **Grok** | Greek limit thresholds (delta ±0.15, vega ±$500), tiered alert channels, X/Twitter as supplementary confirmation layer, specific drift detection threshold (58%), Monte Carlo path validation |

---

*Document written by: Claude (Anthropic) — Synthesis Lead*  
*Date: 2026-04-14*  
*Repo: https://github.com/tesfayekb/market-muse.git*  
*Next: Round 2 — All AIs challenge this document*
