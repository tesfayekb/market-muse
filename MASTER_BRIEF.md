# MASTER BRIEF — AI-Powered Options Trading System
**Version:** 1.0 — Initial Brainstorm Release  
**Status:** Open for AI Review & Proposals  
**Purpose:** This document is the single source of truth for all AI contributors. Read it fully before proposing ideas. Respond to ALL six pillars. Be specific, creative, and ambitious.

---

## ⚠️ INSTRUCTIONS FOR ALL AI CONTRIBUTORS

1. Read this entire document before responding
2. Propose your best architecture and logic for **each of the 6 pillars**
3. Identify what you believe is the **single highest-leverage innovation** in the system
4. Flag any **risks, blind spots, or gaps** you see in the current plan
5. Be specific — no generic advice. Assume a senior quant engineer is reading your response
6. Save your response in `/docs/ai-branches/[your-name]-branch.md`

---

## 1. PROJECT OVERVIEW

### Mission
Build the **most profitable, adaptive, and intelligent options trading system in the world** — one that continuously learns, dynamically adjusts, and executes with precision across changing market conditions.

### Core Philosophy
- **Profitability is the only metric that matters** — every design decision must be justified by its contribution to net P&L
- **Adaptability over rigidity** — the system must perform across different market regimes, not just backtest well on historical data
- **AI is the edge** — not just a feature. AI must be deeply embedded in every layer: prediction, strategy selection, risk management, execution, and learning
- **Discipline beats genius** — the best signal in the world means nothing without iron-clad risk management and exit logic

### What Has Already Been Built
The following infrastructure is **complete and will not be redesigned**:
- User authentication system
- Role-based access control (RBAC)
- Base site structure and frontend scaffolding
- Broker integration: **Tradier API** (execution layer)

All proposals should build **on top of** this existing foundation.

---

## 2. INSTRUMENT UNIVERSE & TAX STRUCTURE

### Primary Instruments
- **SPX options** (S&P 500 Index options — cash settled, European style)
- **XSP options** (Mini SPX — same structure, smaller notional)
- **NDX options** (Nasdaq 100 Index options)
- **RUT options** (Russell 2000 Index options)
- Consideration for **broad-based index ETF options** (/ES, /NQ futures options) where applicable

### Why These Instruments (Non-Negotiable)
- **Section 1256 contracts** — qualify for the **60/40 tax treatment**: 60% taxed as long-term capital gains, 40% as short-term, regardless of actual holding period
- **Cash settlement** — no early assignment risk, cleaner P&L modeling
- **European style** — eliminates early exercise complexity
- **Deep liquidity** — tight bid/ask spreads, minimal slippage, large open interest
- **0DTE availability every trading day** — SPX has Monday/Wednesday/Friday expirations plus daily options

### Tax Alpha Note
The 60/40 structure provides a **structural edge over equity options traders** that must be quantified and baked into the P&L model. At maximum federal rates, this can represent 5–10% additional net return annually versus equivalent equity options positions. The system's P&L engine must model after-tax returns, not just gross P&L.

---

## 3. HOLDING PERIOD STRATEGY

### Primary Mode: 0DTE (Zero Days to Expiration)
- Default operating mode
- Open and close positions within the same trading session
- Exploits maximum theta decay on expiration day
- SPX 0DTE available: **Monday, Wednesday, Friday** (plus daily on some strikes)
- Requires: intraday prediction model, fast execution, tight stop logic, real-time Greeks monitoring

### Secondary Mode: Short Swing (1–5 Days)
- Activated when regime detector identifies a strong multi-day directional or volatility setup
- Captures earnings moves, macro events, trend continuation plays
- Requires: multi-day prediction model, overnight gap risk management, wider stop logic

### Governing Logic: Regime-Gated Mode Selection
**The system, not the user, decides which mode is active each day.**

Regime detector inputs that influence mode selection:
- VIX level and term structure (contango vs backwardation)
- Realized vs implied volatility spread
- Market breadth indicators
- Pre-market futures movement
- Macro calendar (Fed days, CPI, NFP, earnings)
- Previous day's trend continuation probability score

**Key Rule:** On high-uncertainty event days (Fed decisions, CPI prints, major earnings), the system may elect to **reduce size, delay entry, or sit out entirely**. Cash is a position. Sitting out is a valid and sometimes optimal strategy.

---

## 4. CAPITAL ALLOCATION: DYNAMIC TIERED STRUCTURE

### Structure
The system uses a **three-tier dynamic capital allocation model**:

```
CORE POSITION (base: 50–65% of deployed capital)
└── Highest conviction trade of the session
└── Fully vetted by all prediction models
└── All 6 pillars applied with maximum rigor
└── Strict, pre-defined SL and TP levels

SATELLITE POSITIONS (base: 20–35% of deployed capital)
└── Maximum 2 simultaneous satellite trades
└── Opportunistic — momentum, volatility events, secondary setups
└── Smaller size, faster in/out, lower conviction threshold
└── Independent stop-loss from core

RESERVE (base: 10–15% of total account)
└── Never deployed except for exceptional setups
└── Acts as margin buffer and drawdown protection
└── Hard rule: Reserve cannot be touched if daily drawdown limit is hit
```

### Dynamic Allocation Table
Allocation shifts based on the system's **Regime Confidence Score (RCS)** — a 0–100 score output by the regime detection engine:

| RCS Score | Market Condition | Core | Satellites | Reserve |
|---|---|---|---|---|
| 80–100 | High conviction, clear regime | 65% | 25% | 10% |
| 60–80 | Moderate conviction | 50% | 25% | 25% |
| 40–60 | Low conviction / mixed signals | 30% | 10% | 60% |
| 20–40 | Pre-event uncertainty | 20% | 5% | 75% |
| 0–20 | Danger zone / extreme conditions | 0% | 0% | 100% |

### Hard Daily Loss Limit
- If total portfolio drawdown reaches **-3% of account value** in a single session, ALL positions are closed immediately and the system halts trading for the remainder of that day — **no exceptions, no overrides**
- This rule is hardcoded and cannot be disabled by any user role

---

## 5. THE SIX PILLARS — DETAILED REQUIREMENTS

All AI contributors must propose specific solutions for each pillar.

---

### PILLAR 1: Market Movement Prediction Engine

**Goal:** Generate the most accurate possible prediction of SPX/NDX/RUT directional movement, magnitude, and timing across intraday and multi-day horizons.

**Required Outputs:**
- Direction probability (bullish / bearish / neutral) with confidence score
- Expected magnitude (in points and percentage)
- Timing window (when the move is most likely to begin and complete)
- Volatility forecast (expected realized vol vs current implied vol)

**Data Inputs to Consider:**
- Price action and technical indicators (OHLCV, moving averages, momentum oscillators)
- Options flow data (unusual options activity, dark pool prints, put/call ratios)
- Order flow and market microstructure data
- Volatility surface data (IV across strikes and expirations)
- Macro data feeds (economic calendar, Fed watch probabilities, yield curve)
- Sentiment indicators (Fear & Greed, AAII sentiment, news sentiment NLP)
- Cross-asset signals (bonds, dollar, commodities, VIX futures)
- Pre-market futures and overnight action

**AI Contributor Questions:**
- What model architecture do you recommend for each time horizon?
- How should conflicting signals across timeframes be resolved?
- How do you handle model degradation after structural market changes (COVID, GFC-type events)?
- What is your proposed feature engineering pipeline?
- How do you prevent overfitting on historical options data?

---

### PILLAR 2: Strategy Selection Engine

**Goal:** Given the prediction engine output, select the optimal options strategy that maximizes expected P&L while respecting risk parameters.

**Strategy Universe (propose additions or removals):**

| Strategy | Best Condition | Risk Profile |
|---|---|---|
| Long Call / Put | High conviction directional | Defined risk, high reward |
| Vertical Spread (debit) | Moderate directional, defined risk | Defined risk/reward |
| Iron Condor | Low vol, range-bound | Defined risk, premium capture |
| Iron Butterfly | Very low vol, pinning expected | High premium, narrow profit zone |
| Credit Spread | Directional with premium capture | Defined risk |
| Calendar Spread | Volatility differential play | Complex Greeks |
| Straddle / Strangle | High vol expected, direction uncertain | Defined risk, requires big move |
| Broken Wing Butterfly | Directional bias with defined risk | Asymmetric payoff |
| Ratio Spread | Strong directional conviction | Undefined risk on one side |

**Strategy Selection Logic Must Consider:**
- Current regime (trending, ranging, volatile, event-driven)
- Days to expiration (0DTE vs multi-day)
- Current IV rank and IV percentile
- Expected move vs actual options pricing
- Greeks profile (target delta, gamma exposure, theta capture rate, vega exposure)
- Liquidity of specific strikes (bid/ask spread, open interest, volume)

**AI Contributor Questions:**
- How should the system score and rank competing strategy candidates?
- What is your recommended decision tree or ML approach for strategy selection?
- How does strategy selection change between 0DTE and multi-day modes?
- How do you incorporate IV rank into strategy selection without overfitting?

---

### PILLAR 3: Risk Management Engine

**Goal:** Ensure no single trade, day, or sequence of trades can cause catastrophic account damage, while maximizing capital efficiency.

**Required Risk Controls:**

**Position Level:**
- Maximum position size as % of account (dynamic, based on RCS)
- Greeks limits per position (max delta, max gamma, max vega exposure)
- Defined entry criteria — no entry without confirmed stop and target levels
- Correlation check — no two positions with correlated directional exposure above threshold

**Portfolio Level:**
- Maximum total portfolio delta at any time
- Maximum total vega exposure (volatility risk)
- Intraday drawdown limit (-3% hard stop as defined above)
- Maximum number of simultaneous open positions (core + 2 satellites = 3 max)

**Systemic Risk Controls:**
- Pre-market risk scan before any trades are initiated
- Real-time margin utilization monitoring
- Black swan detection: if VIX spikes >15% intraday, all positions reviewed immediately
- Circuit breaker: system-wide halt if SPX moves >2% within 30 minutes of open

**AI Contributor Questions:**
- What is your recommended position sizing formula? (Kelly criterion, fixed fractional, volatility-adjusted?)
- How should the system handle a position that moves against it quickly — scale out, hold, or cut immediately?
- What Greek limits do you recommend for a 0DTE portfolio specifically?
- How do you model tail risk in a way that doesn't over-restrict position sizing?

---

### PILLAR 4: Monitoring & Dashboard Engine

**Goal:** Provide real-time visibility into every dimension of the system's performance and current exposure, enabling rapid human oversight and intervention if needed.

**Required Real-Time Displays:**
- Live P&L by position and total portfolio
- Greeks dashboard (delta, gamma, theta, vega — per position and portfolio total)
- Live prediction confidence scores with model agreement indicator
- Regime status indicator with confidence level
- Capital allocation visualization (core / satellite / reserve utilization)
- Margin utilization and available buying power
- Win rate, average win, average loss, expectancy (rolling 5, 20, 60 day)
- Daily drawdown meter with hard stop proximity alert
- Alert system for approaching SL levels, unusual market moves, regime changes

**AI Contributor Questions:**
- What is the most critical real-time metric that most systems miss?
- How should alerts be tiered (info vs warning vs critical)?
- What visualization best communicates current risk exposure to a non-quant user?
- Should there be an AI assistant embedded in the dashboard that explains what the system is doing and why?

---

### PILLAR 5: P&L, Stop-Loss & Exit Strategy Engine

**Goal:** Maximize captured profit on winning trades and minimize losses on losing trades through intelligent, dynamic exit logic — not static rules.

**Entry & Exit Logic Framework:**

**Pre-Trade:**
- Define profit target (PT) and stop-loss (SL) before every entry
- PT and SL levels output by the prediction + risk engine, not set manually
- Record: entry price, Greeks at entry, prediction score at entry, regime state at entry

**During Trade (Active Management):**
- Trailing stop logic: tighten stop as profit grows (protect profits)
- Partial profit taking: take 50% off at first target, let remainder run with tightened stop
- Greeks-based exits: exit if delta or gamma exceeds threshold (not just price-based)
- Time-based exits: 0DTE positions must be closed by 3:45 PM EST regardless of P&L
- Momentum-based exits: if prediction model confidence drops below threshold, exit regardless of P&L level

**Post-Trade:**
- Full trade audit log: entry/exit price, Greeks, prediction score, regime state, P&L
- Attribution analysis: was the outcome driven by direction, volatility, or time decay?
- Feed results back into the learning engine (Pillar 6)

**Stop-Loss Strategy:**
- Hard stops: price-based, non-negotiable
- Soft stops: Greeks-based or prediction-score-based, requires confirmation
- Time stops: 0DTE mandatory close at 3:45 PM EST
- Portfolio stops: -3% daily drawdown = full halt

**AI Contributor Questions:**
- What is the optimal trailing stop methodology for 0DTE credit spreads specifically?
- How should partial profit taking be timed — at % profit, at time, or at Greeks threshold?
- Should the system ever average into a losing position? Under what conditions, if any?
- How do you model the optimal exit time on a 0DTE trade as theta accelerates into close?

---

### PILLAR 6: AI Learning & Adaptation Engine

**Goal:** Build a system that gets measurably better with every trade, every day — learning from wins and losses to continuously improve prediction accuracy, strategy selection, and risk calibration.

**Learning Architecture:**

**Trade-Level Learning:**
- Every closed trade generates a labeled training record
- Features: market conditions at entry, prediction scores, strategy selected, Greeks profile, regime state, outcome (P&L, win/loss, slippage)
- Models retrain on rolling window of recent trades (not just all historical data)

**Regime-Level Learning:**
- System tracks which strategies outperform in which regimes
- Builds a regime-strategy performance matrix that updates in real time
- Adjusts strategy selection weights based on recent regime performance

**Prediction Model Adaptation:**
- Detects when prediction accuracy drops (model drift detection)
- Triggers retraining or model switching when performance degrades
- Maintains multiple model versions with ensemble voting

**Meta-Learning:**
- The system learns not just *what* to trade but *when NOT to trade*
- Identifies time-of-day, day-of-week, and macro calendar patterns that historically produce poor results
- Builds a "sit out" confidence score that gets smarter over time

**AI Contributor Questions:**
- What is your recommended architecture for the online/incremental learning loop?
- How do you prevent the system from overfitting to recent market conditions during retraining?
- How should the system handle a regime it has never seen before (truly novel market conditions)?
- What is the minimum number of trades needed before the learning engine has statistical significance?
- How do you distinguish between a strategy that is underperforming vs a regime that has changed?

---

## 6. TECHNICAL ARCHITECTURE CONSTRAINTS

### Existing Stack
- Frontend: Already scaffolded (Lovable-built, React-based)
- Auth & RBAC: Complete
- Broker: Tradier API (execution)

### Proposed Additional Stack (open to AI input)
- **Backend:** Python / FastAPI (recommended — propose alternatives)
- **Prediction Models:** Python ML stack (scikit-learn, XGBoost, PyTorch, or similar)
- **Real-time Data:** Market data feed (propose best option for SPX options data)
- **Database:** Propose best solution for time-series trade data + ML feature storage
- **Deployment:** Cloud-based (propose architecture)
- **Scheduling:** Trade session management, pre-market scans, EOD reporting

### Data Requirements
The system requires real-time and historical access to:
- SPX/NDX/RUT options chains (full chain, real-time Greeks)
- Level 2 quotes for target strikes
- VIX real-time and futures curve
- Economic calendar and event data
- Historical options data for backtesting (minimum 5 years)

**AI Contributor Question:** What data providers do you recommend for each data type, balancing cost and quality?

---

## 7. SUCCESS METRICS

The system will be evaluated against these metrics, in priority order:

1. **Net After-Tax P&L** — bottom line, after the 60/40 tax benefit applied
2. **Sharpe Ratio** — risk-adjusted return (target: >2.0)
3. **Maximum Drawdown** — worst peak-to-trough loss (target: <10% of account)
4. **Win Rate** — % of trades closed profitably (target: >60% for premium strategies)
5. **Profit Factor** — gross profit / gross loss (target: >1.8)
6. **Average Win / Average Loss Ratio** — target: >1.5
7. **Prediction Accuracy** — directional accuracy of prediction engine (target: >58%)
8. **Strategy Selection Accuracy** — % of trades where selected strategy was optimal given outcome
9. **Model Drift Response Time** — how quickly the system detects and corrects degrading performance

---

## 8. WHAT WE ARE NOT BUILDING (OUT OF SCOPE)

To keep focus and avoid scope creep, the following are explicitly out of scope for Version 1:

- Cryptocurrency options
- International/non-US markets
- High-frequency trading (sub-second execution)
- Fully automated trading without human oversight (V1 requires human approval for trades)
- Social/copy trading features
- Retail-facing product (this is a proprietary trading tool)

---

## 9. OPEN QUESTIONS FOR ALL AI CONTRIBUTORS

Beyond the pillar-specific questions above, please address:

1. **What is the single most important thing we could do to make this system more profitable than any competitor?**
2. **What is the single biggest risk that could cause this system to fail catastrophically?**
3. **What data source or signal are most trading systems ignoring that you would prioritize?**
4. **How should the system behave during a flash crash or circuit-breaker event?**
5. **What does Version 2 of this system look like — what features are most valuable after V1 is live?**

---

## 10. RESPONSE FORMAT FOR AI CONTRIBUTORS

Please structure your response in `/docs/ai-branches/[your-name]-branch.md` as follows:

```
# [AI Name] — Branch Proposal
**Date:** [Date]
**Reviewing Version:** MASTER_BRIEF v1.0

## My Highest-Leverage Innovation Proposal
[Your single best idea — the thing that would have the most impact on profitability]

## Pillar 1: Prediction Engine Proposal
[Your specific architecture and approach]

## Pillar 2: Strategy Selection Proposal
[Your specific architecture and approach]

## Pillar 3: Risk Management Proposal
[Your specific architecture and approach]

## Pillar 4: Monitoring & Dashboard Proposal
[Your specific architecture and approach]

## Pillar 5: P&L, SL & Exit Strategy Proposal
[Your specific architecture and approach]

## Pillar 6: AI Learning & Adaptation Proposal
[Your specific architecture and approach]

## Technical Stack Recommendations
[Your specific technology recommendations]

## Critical Risks & Blind Spots I See
[What could go wrong that the brief doesn't address]

## Open Questions Responses
[Answers to the 5 open questions in Section 9]
```

---

*Document maintained by: Project Owner*  
*Last updated: 2026*  
*Next update: After AI synthesis round is complete*
