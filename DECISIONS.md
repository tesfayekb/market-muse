# DECISIONS.md — Locked Architectural Decisions
**Repo:** https://github.com/tesfayekb/market-muse.git  
**Purpose:** Every major decision made in this project is recorded here with full reasoning.  
**Rule:** Once a decision is locked, NO AI may re-argue it. Build on it or propose a formal amendment below.

---

## HOW TO USE THIS FILE

- **Locked decisions** — cannot be reopened without owner approval
- **Open decisions** — still being evaluated, AIs may propose
- **Amended decisions** — previously locked, formally revised with new reasoning

If you believe a locked decision should be changed, add to the **Amendment Proposals** section at the bottom. Do not modify the locked entry directly.

---

## LOCKED DECISIONS

---

### DECISION-001: Primary Instrument Universe
**Status:** 🔒 LOCKED  
**Date Locked:** 2026  
**Owner:** tesfayekb

**Decision:** The system will trade exclusively on Section 1256 broad-based index options:
- SPX (S&P 500 Index options)
- XSP (Mini SPX)
- NDX (Nasdaq 100 Index options)
- RUT (Russell 2000 Index options)

**Reasoning:**
- Section 1256 contract status provides 60/40 long-term/short-term tax treatment regardless of holding period
- At maximum federal tax rates this represents a 5–10% structural annual return advantage over equivalent equity options strategies — this is free alpha that must be preserved
- European-style cash settlement eliminates early assignment risk and simplifies P&L modeling
- Deep liquidity ensures tight bid/ask spreads and minimal execution slippage

**Rejected Alternatives:**
- Equity options (SPY, QQQ, individual stocks) — lose Section 1256 tax treatment
- Crypto options — out of scope, unregulated, no tax advantage
- Futures options (/ES, /NQ) — may be added in V2 if liquidity/margin analysis supports it

---

### DECISION-002: Primary Trading Mode
**Status:** 🔒 LOCKED  
**Date Locked:** 2026  
**Owner:** tesfayekb

**Decision:** 0DTE (zero days to expiration) is the default and primary operating mode.

**Reasoning:**
- Maximum theta decay occurs on expiration day — premium sellers capture the steepest part of the decay curve
- SPX now offers 0DTE opportunities every trading day
- Most capital-efficient use of margin — positions open and close same session
- No overnight gap risk in primary mode

**Rejected Alternatives:**
- Multi-week holds — capital inefficient, more black swan exposure
- Pure swing trading — loses theta advantage

**Constraints This Decision Creates:**
- Prediction engine must have strong intraday (1-min to 30-min) forecasting capability
- All 0DTE positions must be closed by 3:45 PM EST — hardcoded rule
- Gamma risk management is critical — 0DTE gamma can be explosive near expiration

---

### DECISION-003: Secondary Trading Mode
**Status:** 🔒 LOCKED  
**Date Locked:** 2026  
**Owner:** tesfayekb

**Decision:** Short swing (1–5 days) is the secondary mode, activated by regime detector only.

**Reasoning:**
- Captures multi-day momentum, macro event setups, and earnings moves that 0DTE cannot
- Provides diversification of alpha sources — not all profit from theta decay
- Regime detector prevents activation on low-conviction days

**Key Rule:** The system decides mode — not the user. Manual mode override is not permitted in V1.

---

### DECISION-004: Capital Allocation Structure
**Status:** 🔒 LOCKED  
**Date Locked:** 2026  
**Owner:** tesfayekb

**Decision:** Dynamic three-tier capital allocation — Core Position + Satellites + Reserve.

**Base Allocation:**
- Core: 50–65% of deployed capital
- Satellites: 20–35% of deployed capital (max 2 simultaneous)
- Reserve: 10–15% of total account (never deployed except exceptional setups)

**Dynamic Adjustment:** Allocation shifts based on Regime Confidence Score (RCS) — see MASTER_BRIEF Section 4 for full table.

**Reasoning:**
- Single position caps upside and leaves capital idle
- Multiple simultaneous positions without tiering creates uncontrolled correlated risk
- Tiered structure maximizes at-bats while preventing any single position from being catastrophic
- Dynamic reserve ensures margin buffer and drawdown protection at all times

**Rejected Alternatives:**
- Fixed single position — too capital inefficient
- Equal-weight multi-position — no priority weighting, correlation risk unmanaged

---

### DECISION-005: Hard Daily Loss Limit
**Status:** 🔒 LOCKED  
**Date Locked:** 2026  
**Owner:** tesfayekb

**Decision:** If total portfolio drawdown reaches -3% of account value in any single session, ALL positions are closed immediately and the system halts trading for the remainder of that day.

**Reasoning:**
- Prevents a bad day from becoming a catastrophic week
- Removes emotional decision-making from loss situations
- -3% is aggressive enough to allow normal trading but tight enough to prevent compounding losses

**Implementation Rule:** This rule is hardcoded. It cannot be disabled, overridden, or modified by any user role including admin. It can only be changed by modifying source code with owner approval.

**Rejected Alternatives:**
- -5% limit — too loose, allows too much single-day damage
- No limit — unacceptable
- User-configurable limit — too much risk of override under emotional pressure

---

### DECISION-006: Execution Broker
**Status:** 🔒 LOCKED  
**Date Locked:** 2026  
**Owner:** tesfayekb

**Decision:** Tradier API is the sole execution broker for V1.

**Reasoning:**
- Already integrated
- Supports SPX/NDX/RUT options trading
- Competitive commission structure
- Reliable API with documented endpoints

**Note for AIs:** All execution proposals must be compatible with Tradier API capabilities and constraints. See `/docs/constraints/broker_constraints.md` for specific limits.

---

### DECISION-007: Human Approval Requirement (V1)
**Status:** 🔒 LOCKED  
**Date Locked:** 2026  
**Owner:** tesfayekb

**Decision:** V1 requires human approval before trade execution. The system recommends; the human approves.

**Reasoning:**
- Regulatory and risk prudence for initial deployment
- Allows validation of AI recommendations against real market conditions before full automation
- Builds confidence in the system before removing human oversight

**Future State:** Full automation is the V2 target once the system has demonstrated consistent accuracy over a sufficient live trading sample.

---

## OPEN DECISIONS (AIs May Propose)

---

### OPEN-001: Prediction Model Architecture
**Status:** 🔵 OPEN  
**Question:** What specific ML architecture(s) should power the prediction engine for 0DTE intraday forecasting?  
**Leading candidates:** Ensemble (XGBoost + LSTM), Transformer-based, Pure gradient boosting  
**Profit constraint:** Must demonstrate >58% directional accuracy in backtest with realistic transaction costs

---

### OPEN-002: Data Provider Selection
**Status:** 🔵 OPEN  
**Question:** Which data providers should be used for real-time options chain data, historical options data, and alternative signals?  
**Profit constraint:** Must balance data quality with cost — data cost directly reduces net P&L

---

### OPEN-003: Regime Classification Framework
**Status:** 🔵 OPEN  
**Question:** What are the specific regime types the system should recognize, and what signals define each regime?  
**Leading candidates:** 4-regime model (trending/ranging/volatile/event-driven) vs. continuous RCS score  
**Profit constraint:** Regime misclassification directly causes wrong strategy selection — accuracy is critical

---

### OPEN-004: Position Sizing Formula
**Status:** 🔵 OPEN  
**Question:** What position sizing methodology maximizes long-run capital growth while respecting drawdown limits?  
**Leading candidates:** Kelly Criterion (modified), fixed fractional, volatility-adjusted sizing  
**Profit constraint:** Oversizing destroys accounts. Undersizing destroys returns. Must be calibrated precisely.

---

### OPEN-005: Learning Engine Architecture
**Status:** 🔵 OPEN  
**Question:** What is the optimal architecture for the continuous learning loop — online learning, rolling window retraining, or ensemble versioning?  
**Profit constraint:** Model drift without correction is a guaranteed path to losses

---

## AMENDMENT PROPOSALS

*No amendments proposed yet. Add here if you believe a locked decision should be revisited.*

**Format:**
```
### AMENDMENT PROPOSAL — DECISION-XXX
**Proposed by:** [AI Name]
**Date:** [Date]
**Argument for amendment:** [Specific profit-based reasoning]
**Evidence:** [Backtest data, research, or concrete analysis]
```

---

*Maintained by: tesfayekb | Repo: https://github.com/tesfayekb/market-muse.git*
