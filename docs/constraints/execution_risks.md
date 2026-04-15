# execution_risks.md — Execution Risk Factors
**Repo:** https://github.com/tesfayekb/market-muse.git  
**Path:** `/docs/constraints/execution_risks.md`  
**Purpose:** Documents every known execution risk that can destroy profit. Every AI proposal must account for these.

---

## ⚠️ RULE FOR ALL AI CONTRIBUTORS

A perfect signal with poor execution is a losing system. Read every item in this file before proposing execution, exit, or risk management logic.

---

## 1. SLIPPAGE — THE SILENT PROFIT KILLER

### What It Is
The difference between your expected fill price and your actual fill price.

### SPX 0DTE Reality
On a $2.00 mid-price option, realistic slippage scenarios:

| Market Condition | Expected Slippage Per Leg | Round-Trip (Entry + Exit) |
|---|---|---|
| Calm midday market | $0.05–$0.10 | $0.10–$0.20 |
| Moderate volatility | $0.15–$0.30 | $0.30–$0.60 |
| High volatility / news | $0.50–$1.50 | $1.00–$3.00 |
| Market open (9:30–10:00) | $0.50–$2.00 | $1.00–$4.00 |

**Profit Impact Example:**
- Iron condor credit: $1.50
- Round-trip slippage at $0.40: effective credit drops to $1.10 — a 27% reduction in max profit before the trade even starts

**System Requirement:** All P&L modeling must include realistic slippage assumptions. Backtests that assume mid-price fills are fantasy.

### Slippage Reduction Strategies (Required in Proposals)
- Use limit orders placed at or slightly better than mid-price
- Never chase a fill — if not filled within 30 seconds, cancel and reassess
- Avoid trading in the first 30 minutes (spreads are widest)
- Target high open-interest strikes — tighter spreads, better fills

---

## 2. GAMMA RISK ON 0DTE — THE ACCOUNT DESTROYER

### What It Is
Gamma measures how fast delta changes as price moves. On 0DTE, gamma is at its absolute peak near ATM strikes.

### The Risk
A position that looks safe with a 0.20 delta at 10am can have a 0.70 delta by 2pm if price hasn't moved — purely from time compression of gamma.

### Quantified Risk
On a typical 0DTE SPX credit spread:
- Entry: Delta 0.20, Gamma 0.05 (manageable)
- 2 hours later, same position: Delta 0.45, Gamma 0.15 (dangerous)
- 1 hour before close, ITM: Delta 0.85, Gamma 0.40 (catastrophic)

**System Requirement:** Greeks-based exit triggers must be built into every position. A position must be exited if gamma exceeds a defined threshold — not just if price moves against you.

---

## 3. LIQUIDITY EVAPORATION RISK

### What It Is
Options that had good liquidity at entry can become nearly illiquid by close, making exit fills extremely poor.

### When It Happens
- Deep ITM or OTM strikes after a large move
- Any 0DTE position held past 3:30 PM EST
- Low-volume strikes that looked liquid at open but have thin markets

### System Requirement
- Mandatory close of all 0DTE by 3:45 PM EST — hardcoded, no exceptions
- Strike selection criteria must include minimum open interest and volume thresholds (see broker_constraints.md)
- Exit routing must attempt limit order first, then widen by $0.05 increments every 10 seconds to ensure fill

---

## 4. GAP RISK (SWING POSITIONS ONLY)

### What It Is
Overnight or weekend price gaps that open beyond your stop-loss level, causing fills far worse than intended.

### SPX Historical Gap Data
- Average overnight gap: 0.2–0.4%
- Significant gap (>1%): occurs ~15% of trading days
- Extreme gap (>2%): occurs ~3–5% of trading days, almost always macro-event driven

### System Requirement for Swing Positions
- Maximum position size in swing mode must be smaller than 0DTE — accounts for gap risk
- Pre-market check every morning: if gap exceeds defined threshold, review all swing positions before market open
- Swing positions must never be held through scheduled high-impact events (FOMC, CPI, NFP) without explicit risk-reduction sizing

---

## 5. VOLATILITY EXPANSION RISK (VEGA EXPOSURE)

### What It Is
A position that looks profitable on direction can lose money if implied volatility expands unexpectedly.

### Example
You sell an iron condor expecting low volatility. VIX spikes 20% intraday after an unexpected headline. The options you sold are now worth significantly more due to vega — not because price moved, but because fear increased.

### System Requirement
- Portfolio-level vega exposure limit must be defined and enforced
- Vega exposure must be monitored in real time
- On high-VIX days or pre-event days, vega-sensitive strategies (short premium) must be sized down or avoided entirely

---

## 6. CORRELATED POSITION RISK

### What It Is
Holding multiple positions that all lose together in the same market move.

### The Illusion
Holding SPX + NDX + RUT positions feels like diversification. In a sharp market sell-off, all three move down simultaneously with high correlation (0.80–0.95 correlation during stress events).

### System Requirement
- No two positions should have correlated directional exposure above a defined threshold
- Portfolio delta must be monitored as a net number across ALL positions
- If net portfolio delta exceeds defined threshold, no new directional positions may be opened in the same direction

---

## 7. MODEL CONFIDENCE VS MARKET REALITY

### What It Is
The prediction engine outputs a confidence score — but confidence is not certainty. A 75% confidence bullish signal fails 25% of the time.

### The Sequence Risk
After 3 consecutive losses on 75% confidence signals, the system may be in a regime where the model is less accurate than expected. This is model drift.

### System Requirement
- Track model accuracy on a rolling basis (last 20 trades, last 60 trades)
- If rolling accuracy drops more than 10 percentage points below historical average, trigger model drift alert
- On model drift alert: reduce position sizes automatically until accuracy recovers
- Never increase position size after losses — the instinct to "make it back" is the most common account destruction pattern

---

## 8. FLASH CRASH / CIRCUIT BREAKER PROTOCOL

### Scenarios
| Event | Trigger | System Response |
|---|---|---|
| SPX drops >2% in 30 min | Circuit breaker risk | Close all positions immediately, halt new entries |
| VIX spikes >15% intraday | Volatility shock | Review all positions, tighten stops, reduce size |
| Market-wide circuit breaker (Level 1: -7%) | Exchange halt | All orders cancelled, system halts, await human decision |
| Market-wide circuit breaker (Level 2: -13%) | Exchange halt | Same as Level 1, mandatory human review before restart |
| Market-wide circuit breaker (Level 3: -20%) | Market closes | System halts for remainder of day, human review required |

### System Requirement
- Real-time SPX % move calculation running continuously during market hours
- VIX level monitored every 60 seconds
- Automatic position closure logic must execute in <5 seconds of trigger
- Human notification (email/SMS/push) must be sent on any circuit breaker trigger

---

## 9. EXECUTION FAILURE MODES

### Known Failure Scenarios
| Failure | Cause | Mitigation |
|---|---|---|
| Order rejected | Insufficient margin, invalid strike | Retry with adjusted size, alert user |
| Partial fill | Low liquidity | Handle partial state, decide to complete or cancel remainder |
| API timeout | Network issue | Retry with exponential backoff, max 3 retries |
| Fill confirmation delayed | Exchange processing lag | Set maximum wait time, query order status after 10 seconds |
| Stale quote data | Feed interruption | Detect gap in streaming data, pause new orders until feed restored |
| Duplicate order | Retry on partial fill | Implement idempotency keys on all order submissions |

### System Requirement
- Every order submission must have a unique idempotency key
- All API calls must have timeout handling (max 5 seconds)
- Order state machine must handle: pending → submitted → partial → filled → cancelled
- Any execution failure must be logged with full context for post-session review

---

## 10. THE COSTS THAT AIs ALWAYS FORGET

Every P&L model must include ALL of these:

| Cost | Frequency | Estimated Impact |
|---|---|---|
| Bid/ask slippage | Every trade | $0.10–$1.00 per contract per leg |
| Commission (Tradier) | Every trade | ~$0.35 per contract (check current Tradier pricing) |
| Regulatory fees | Every trade | Minimal but non-zero |
| Margin interest | Overnight positions | Based on account size and broker rate |
| Data feed costs | Monthly | $50–$500/month depending on providers |
| Infrastructure (cloud) | Monthly | $100–$500/month for production system |

**Net P&L = Gross P&L − Slippage − Commissions − Fees − Infrastructure Costs**

A system that shows 15% gross return but costs 5% in friction is a 10% net system — not a 15% system. All backtests and live performance metrics must use net figures.

---

*Maintained by: tesfayekb | Repo: https://github.com/tesfayekb/market-muse.git*
