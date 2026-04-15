# Claude — Branch Proposal

**Date:** 2026-04-15  
**Reviewing Version:** MASTER_BRIEF v1.0

---

## My Highest-Leverage Innovation Proposal

### Gamma-Aware Microstructure Execution with Real-Time Dealer Positioning Model

The single highest-impact innovation is building a **dealer gamma exposure (GEX) model** that runs in real-time and directly feeds into both the prediction engine and exit logic. Here's why this is the edge:

Market makers who sell options to retail and institutional clients accumulate gamma exposure. When dealers are **short gamma**, they must buy into rallies and sell into selloffs — amplifying moves. When dealers are **long gamma**, they do the opposite — dampening volatility.

**No retail or mid-tier institutional system models this correctly in real-time.** Most systems treat the market as a random walk with drift. But SPX intraday moves are heavily influenced by dealer hedging flows, which are deterministic given knowledge of:

- Open interest distribution across strikes
- Put/call OI ratios at each strike
- Daily volume and OI changes (new positions vs closures)
- The "gamma flip point" — the SPX level where dealer positioning transitions from long to short gamma

**Implementation:**

1. Build a GEX calculator that ingests the full SPX options chain every 5 minutes
2. Compute net dealer gamma at each strike using a dealer positioning heuristic (customers are net long puts, net short calls at OTM strikes)
3. Identify the **gamma flip level** — above this, dealers dampen; below, they amplify
4. Feed GEX into the prediction engine as the highest-weighted signal for 0DTE
5. Use GEX to dynamically adjust stop widths — wider when dealers are short gamma (expect amplified moves), tighter when long gamma

**Expected impact:** 10-15% improvement in 0DTE directional prediction accuracy. This is the signal most competitors don't have because it requires real-time options chain processing and a correct dealer positioning model.

---

## Pillar 1: Prediction Engine Proposal

### Architecture: Hierarchical Temporal Ensemble (HTE)

I propose a three-layer architecture, each optimized for a different prediction horizon:

#### Layer 1: Intraday Microstructure Model (0DTE, 5-min to 4-hour horizon)

**Model:** Temporal Fusion Transformer (TFT) — specifically designed for multi-horizon time series with mixed inputs.

**Why TFT over LSTM/standard transformers:**
- Native handling of static covariates (day of week, expiration type) alongside time-varying inputs
- Built-in variable selection networks — automatically identifies which features matter for each prediction
- Interpretable attention weights — we can see which inputs drove the prediction
- Multi-horizon output — single model predicts 15-min, 30-min, 1-hour, 2-hour, 4-hour simultaneously

**Feature Engineering Pipeline:**

```
RAW DATA
├── Price microstructure (1-min OHLCV, VWAP, tick imbalance)
├── Options flow (GEX as described above, put/call volume ratio, unusual activity flags)
├── Volatility surface (ATM IV, 25-delta skew, term structure slope)
├── Order flow (bid/ask imbalance, large block detection, sweep detection)
└── Cross-asset (ES futures, /NQ, VIX, /ZN 10Y, DXY)

ENGINEERED FEATURES
├── Momentum: ROC(5,15,30,60-min), RSI(14), MACD, Bollinger %B
├── Volatility: Realized vol (5,15,30-min windows), IV-RV spread, VIX term structure slope
├── Microstructure: VWAP deviation, tick direction ratio, cumulative delta
├── Options: GEX level, gamma flip distance, put/call OI skew, max pain level
├── Calendar: Minutes since open, DTE, is_FOMC, is_CPI, is_NFP, is_opex
└── Regime: HMM state probabilities (from Layer 3)
```

**Overfitting Prevention:**
- **Walk-forward validation only** — no standard train/test split. Use expanding window with 6-month minimum training, 1-month validation, 1-week test
- **Feature importance monitoring** — if any feature's importance exceeds 25%, investigate for data leakage
- **Adversarial validation** — train a classifier to distinguish train vs test sets; if accuracy > 55%, data shift is occurring
- **Dropout + weight decay** — standard regularization with 0.3 dropout
- **Ensemble disagreement monitoring** — if ensemble members diverge significantly, reduce confidence score

#### Layer 2: Multi-Day Directional Model (1-5 day horizon)

**Model:** Gradient Boosted Decision Trees (XGBoost/LightGBM) ensemble

**Why trees over neural nets for multi-day:**
- More robust to regime changes with fewer data points
- Handles missing data natively (market holidays, data gaps)
- Faster retraining cycle for the learning engine
- Better calibrated probabilities with Platt scaling

**Features (additional to Layer 1):**
- Daily OHLCV patterns (inside days, outside days, gap fills)
- Breadth indicators (A/D line, new highs/lows, % above 20/50 DMA)
- Macro: yield curve slope, credit spreads (HY-IG), Fed funds futures
- Sentiment: AAII bull/bear ratio, CNN Fear & Greed, options sentiment (put/call ratio 5-day MA)
- Seasonal patterns (turn-of-month, FOMC drift, OpEx week effects)

#### Layer 3: Regime Detection Model

**Model:** Hidden Markov Model (HMM) with 5 states

**States:**
1. **Trending Bull** — strong directional up, low VIX, breadth expanding
2. **Trending Bear** — strong directional down, rising VIX, breadth contracting
3. **Range-Bound Low Vol** — consolidation, VIX < 15, narrow daily ranges
4. **High Volatility Event** — VIX > 25 or VIX spike > 15%, wide daily ranges
5. **Transition/Uncertain** — regime change in progress, signals conflicting

**Outputs:**
- Current state probabilities (soft classification)
- Regime Confidence Score (RCS): `max(state_probabilities) * 100`
- Regime duration estimate (how long current regime is likely to persist)
- Transition probability matrix (which regime is most likely next)

**Signal Conflict Resolution:**
When Layer 1 and Layer 2 disagree:
1. If Layer 3 (regime) confidence > 80%, defer to the model that historically performs best in the detected regime
2. If Layer 3 confidence < 80%, reduce overall prediction confidence by 30% and default to lower-risk strategies
3. Log all conflicts for the learning engine — these are high-value training samples

**Handling Structural Breaks (COVID/GFC events):**
- Maintain a "novelty detection" module using an autoencoder trained on historical feature distributions
- When reconstruction error exceeds 3σ, flag as potential structural break
- In structural break mode: reduce all position sizes by 50%, widen stops by 2x, increase retraining frequency to daily
- After 20 trading days of stable new regime, recalibrate HMM with new data included

---

## Pillar 2: Strategy Selection Proposal

### Architecture: Context-Aware Strategy Scorer (CASS)

Rather than a decision tree (brittle, hard to maintain) or pure ML (black box), I propose a **hybrid scoring system**:

#### Step 1: Generate Candidate Strategies

Given prediction output, pre-filter to 3-5 candidate strategies based on hard rules:

```python
def prefilter(prediction, regime, dte):
    candidates = []
    
    if dte == 0:  # 0DTE mode
        if prediction.direction_confidence > 0.75:
            candidates += [VERTICAL_DEBIT_SPREAD, LONG_OPTION]
        if prediction.direction_confidence < 0.55:
            candidates += [IRON_CONDOR, IRON_BUTTERFLY]
        if prediction.expected_magnitude > 1.5 * implied_move:
            candidates += [STRADDLE, STRANGLE]
        candidates += [CREDIT_SPREAD]  # Always consider
    else:  # Multi-day mode
        if prediction.vol_forecast > current_iv * 1.1:
            candidates += [CALENDAR_SPREAD, STRADDLE]
        if prediction.direction_confidence > 0.70:
            candidates += [VERTICAL_DEBIT_SPREAD, BROKEN_WING_BUTTERFLY]
        candidates += [IRON_CONDOR, CREDIT_SPREAD]
    
    return candidates
```

#### Step 2: Score Each Candidate

Each candidate is scored on 6 dimensions (0-100 each):

| Dimension | Weight | What It Measures |
|-----------|--------|-----------------|
| Expected Value | 30% | `P(profit) * avg_win - P(loss) * avg_loss` using prediction probabilities |
| Risk Efficiency | 20% | Max loss / buying power reduction ratio |
| Greeks Alignment | 15% | How well the strategy's Greeks match the prediction (e.g., long vega if vol expansion predicted) |
| Liquidity Score | 15% | Bid/ask spread at target strikes, OI, volume |
| Regime Fit | 10% | Historical win rate of this strategy in current regime |
| Theta Profile | 10% | For 0DTE: theta capture rate. For multi-day: theta drag cost |

**Composite Score:** `Σ(dimension_score * weight)` — highest score wins.

#### Step 3: Strike Selection Optimizer

Once strategy type is selected, optimize strike placement:

- **Credit spreads:** Target short strike at 1σ (68% probability OTM), width based on max acceptable loss
- **Debit spreads:** Long strike ATM or slightly ITM for delta, short strike at target price level
- **Iron condors:** Wing placement at 1.5σ, balanced premium collection
- **All strategies:** Verify bid/ask spread < 10% of credit received (or debit paid)

#### IV Rank Integration (Without Overfitting)

- IV Rank > 50th percentile: Favor credit strategies (premium is rich)
- IV Rank < 30th percentile: Favor debit strategies (premium is cheap)
- The thresholds are NOT fixed — the learning engine adjusts them based on recent performance
- IV Rank is one input to the scoring system, not a binary gate

---

## Pillar 3: Risk Management Proposal

### Position Sizing: Modified Half-Kelly with Volatility Scaling

**Formula:**

```
base_size = Kelly_fraction / 2
vol_scalar = target_vol / realized_vol_20d
regime_scalar = RCS / 100
position_size = base_size * vol_scalar * regime_scalar * account_value

Kelly_fraction = (win_rate * avg_win_ratio - (1 - win_rate)) / avg_win_ratio
```

**Why Half-Kelly:** Full Kelly is theoretically optimal but assumes perfect knowledge of edge. Half-Kelly sacrifices ~25% of theoretical growth for ~50% reduction in variance — critical for a system that's still learning.

**Why volatility scaling:** Constant-dollar position sizes create variable risk exposure. A $5,000 position in VIX 12 is wildly different from VIX 30.

### Greek Limits (0DTE Specific)

| Greek | Per Position Max | Portfolio Max | Rationale |
|-------|-----------------|---------------|-----------|
| Delta | ±30 | ±50 | SPX point moves of 50+ pts are 2σ events; limits max directional loss |
| Gamma | ±5 | ±8 | Gamma explodes near expiry; this prevents runaway delta |
| Theta | No limit (positive preferred) | No limit | We want theta; it's our edge |
| Vega | ±$500 | ±$1,000 | 0DTE vega is naturally small; this is a safety net |

### Rapid Adverse Move Protocol

When a position moves against by > 50% of max loss within 15 minutes:

1. **Immediate:** Reduce position by 50% (market order)
2. **Reassess:** Run prediction engine with updated data
3. **If prediction still supports the trade:** Hold remainder with tightened stop at 75% of max loss
4. **If prediction has flipped:** Close entire remaining position immediately
5. **Never:** Average into a losing 0DTE position. Multi-day only, and only if prediction confidence increases AND regime hasn't changed.

### Tail Risk Modeling

Use **Extreme Value Theory (EVT)** with Generalized Pareto Distribution to model tail losses:
- Calibrate on daily SPX returns below the 5th percentile
- Compute 99.5% VaR and CVaR for portfolio
- Position sizing constraint: no position whose 99.5% CVaR exceeds 1.5% of account value
- This replaces naive Gaussian VaR which dramatically underestimates tail risk

### Correlation Monitoring

- Maintain a rolling 20-day correlation matrix between all instruments (SPX, NDX, RUT)
- If correlation between any two positions exceeds 0.85, treat them as a single position for sizing purposes
- On high-VIX days (>25), assume correlations go to 1.0 for all instruments (as they empirically do in crises)

---

## Pillar 4: Monitoring & Dashboard Proposal

### The Critical Metric Most Systems Miss

**Gamma Exposure at Current Price (GEX@Spot)**

Most dashboards show P&L, Greeks, and some indicators. Almost none show the market's aggregate gamma positioning relative to spot. This single metric tells you:
- Whether the market is likely to trend or mean-revert today
- Whether your stops are appropriately sized for the current dealer hedging regime
- When a regime change is imminent (GEX flipping sign)

### Dashboard Architecture: Three Panels

#### Panel 1: Cockpit (Always Visible)

```
┌─────────────────────────────────────────────────────┐
│  REGIME: Trending Bull (87% conf)  │  RCS: 84       │
│  GEX: +$2.1B (Long Gamma)         │  Flip: 5,245   │
│  Daily P&L: +$1,240 (+0.62%)      │  DD: -0.3%     │
│  Capital: Core 62% | Sat 23% | Res 15%              │
│  [██████████████████░░] -3% Hard Stop                │
└─────────────────────────────────────────────────────┘
```

#### Panel 2: Position Detail (Expandable per position)

- Entry price, current price, unrealized P&L
- Greeks at entry → Greeks now (delta, gamma, theta, vega)
- Time in trade, prediction confidence at entry vs now
- Stop-loss level, profit target, trailing stop level
- Visual: payoff diagram overlaid with current price marker

#### Panel 3: System Intelligence (Right sidebar)

- Model agreement indicator: 🟢 All aligned / 🟡 Minor divergence / 🔴 Major conflict
- Prediction timeline: next expected move, confidence, timing
- Recent trade history with attribution (direction/vol/theta contribution)
- Learning engine status: last retrain, drift score, model version

### Alert Tiers

| Tier | Trigger | Action | Delivery |
|------|---------|--------|----------|
| 🔵 Info | New prediction, regime update, trade opened/closed | Log only | Dashboard |
| 🟡 Warning | Position at 60% of max loss, IV spike >20%, model confidence drop | Sound + banner | Dashboard + push |
| 🔴 Critical | Position at 80% of max loss, VIX spike >15%, -2% daily drawdown | Alarm + auto-action | Dashboard + push + SMS |
| ⚫ Emergency | -3% drawdown hit, circuit breaker, system error | Full halt + notification | All channels |

### Embedded AI Assistant: YES

An embedded AI assistant is high-value for this system because:
- It can narrate *why* the system is doing what it's doing in plain English
- It can explain regime changes, strategy selections, and exit decisions
- It provides the human oversight layer required in V1
- Implementation: use the same prediction outputs + trade log to generate natural language explanations
- Example: *"Opened core iron condor at 5240/5235/5260/5265. Regime: range-bound (83% conf). GEX is strongly positive, suggesting dealer hedging will pin SPX near 5250. Target: collect 65% of max premium by 2PM. Hard stop at -$800."*

---

## Pillar 5: P&L, SL & Exit Strategy Proposal

### 0DTE Trailing Stop Methodology for Credit Spreads

**The "Theta Curve Trail"** — a trailing stop that adapts to the theoretical theta decay curve:

```python
def trailing_stop_0dte(position, current_time, entry_time):
    # Theta decays non-linearly: slow early, accelerating into close
    hours_remaining = (market_close - current_time).hours
    total_hours = (market_close - market_open).hours
    time_fraction = 1 - (hours_remaining / total_hours)
    
    # Expected profit at this time point (assuming max profit at close)
    expected_theta_capture = max_profit * (time_fraction ** 1.5)  # Power curve
    
    # Trail: protect [captured_profit - cushion]
    cushion = max(0.20, 0.40 * (1 - time_fraction))  # Cushion shrinks over time
    trail_level = current_profit - (max_profit * cushion)
    
    # Floor: never let trail go below entry (breakeven trail after profitable)
    if current_profit > max_profit * 0.30:
        trail_level = max(trail_level, 0)  # Breakeven floor activates at 30% profit
    
    return trail_level
```

**Why this works:** Static trailing stops (e.g., "trail at 50% of max profit") don't account for the non-linear theta curve. Early in the day, you need wider stops because gamma risk is high and the position hasn't captured theta yet. Late in the day, tighten aggressively because most theta is captured.

### Partial Profit Taking Protocol

| Trigger | Action | Rationale |
|---------|--------|-----------|
| 50% of max profit reached | Close 50% of position | Lock in profit; remaining position is "free" |
| 75% of max profit reached | Close another 25% (75% total closed) | Diminishing returns on remaining theta |
| 90% of max profit reached OR 3:30 PM | Close remaining | Don't risk a late reversal for 10% more |
| Any time after 3:00 PM with >60% profit | Close all | Gamma risk outweighs remaining theta |

### Averaging Into Losing Positions

**0DTE: NEVER.** There is no scenario where adding to a losing 0DTE position is justified. The theta you're trying to capture is a known quantity; adding size doesn't increase your per-unit theta, it only increases your gamma exposure on a position that is already losing.

**Multi-day (1-5 DTE): Only under ALL of these conditions simultaneously:**
1. Prediction model confidence has *increased* since entry (new data supports the thesis)
2. Regime has NOT changed
3. Current loss is < 30% of max loss
4. Adding size would not breach any Greek or position sizing limit
5. Maximum one add — no pyramiding of adds

### Optimal Exit Timing Model for 0DTE

Use a **cost-benefit decay function:**

```
hold_value(t) = remaining_theta(t) - gamma_risk(t) - transaction_cost
```

Where:
- `remaining_theta(t)` = premium left to capture * P(staying OTM)
- `gamma_risk(t)` = |gamma| * expected_move_remaining(t) * position_size
- `transaction_cost` = estimated slippage + commission

**Exit when `hold_value(t) < 0`** — the cost of holding exceeds the expected benefit.

In practice, this usually triggers between 3:00-3:30 PM for winning credit spreads. The exact time depends on how close spot is to the short strike.

---

## Pillar 6: AI Learning & Adaptation Proposal

### Architecture: Three-Speed Learning Loop

#### Speed 1: Real-Time Adaptation (Intraday)

- **What updates:** Prediction confidence weights, stop levels, position sizing scalars
- **How:** Bayesian updating of prediction probabilities as new data arrives within the session
- **Not updated:** Model weights, strategy selection logic, risk parameters
- **Latency:** Sub-second

#### Speed 2: Daily Batch Learning (End of Day)

- **What updates:** Strategy scoring weights, regime-strategy performance matrix, feature importance rankings
- **How:** After market close, process all trades from the day:
  1. Generate labeled training records
  2. Update rolling performance statistics per strategy per regime
  3. Run model drift detection (compare recent accuracy to baseline)
  4. If drift detected: flag for weekend retraining
- **Storage:** Append to TimescaleDB trade log with full feature snapshot

#### Speed 3: Weekly Model Retraining (Weekend)

- **What updates:** Model weights for TFT, XGBoost, and HMM
- **How:**
  1. Retrain on expanding window (all historical data, weighted toward recent)
  2. Validate on most recent 4 weeks (out-of-sample)
  3. Compare new model vs current model on validation set
  4. **Only deploy if new model improves Sharpe by >0.1 on validation set** — no lateral moves
  5. Keep previous 3 model versions as fallback
- **Overfitting prevention during retraining:**
  - Time-series cross-validation with purge gaps (5-day purge between train/test)
  - Maximum feature count: 50 (force dimensionality discipline)
  - Early stopping on validation loss with patience=10

### Handling Novel Regimes

When the novelty detector (autoencoder) flags a regime it's never seen:

1. **Immediate:** Reduce all position sizes to 50% of normal, increase reserve to max
2. **First 5 days:** Trade at reduced size, log everything, but do NOT retrain yet — insufficient data
3. **Days 5-20:** Begin incorporating new regime data into the HMM, but keep old model as primary
4. **Day 20+:** If new regime data is consistent, allow full retraining with new data included
5. **Key principle:** In novel regimes, the meta-learning "sit out" score should naturally increase

### Minimum Sample Sizes for Statistical Significance

| What You're Measuring | Minimum Trades | Rationale |
|----------------------|----------------|-----------|
| Strategy win rate in a regime | 30 | Binomial proportion CI width < ±15% at 95% confidence |
| Average P&L per strategy | 50 | Need sufficient samples to estimate mean with useful precision |
| Regime detection accuracy | 60 days of regime data | HMM needs multiple state transitions to calibrate |
| Prediction model accuracy | 200 predictions | Multi-class accuracy requires larger samples |
| Full system edge (is it profitable?) | 500 trades | Account for autocorrelation in trade outcomes |

### Distinguishing Underperforming Strategy vs Regime Change

**Two concurrent tests:**

1. **Strategy-specific test:** Compare the strategy's rolling 20-trade win rate to its historical average in the *current* detected regime. If it drops >2σ below the mean, flag the strategy.

2. **Cross-strategy test:** If multiple strategies are underperforming simultaneously, it's likely a regime change, not individual strategy failure. Trigger the HMM to increase its learning rate and re-evaluate state probabilities.

**Decision logic:**
- Only strategy X underperforming → reduce Strategy X weight, investigate
- Multiple strategies underperforming → regime change likely → re-run regime detection with fresh parameters
- Everything underperforming AND novelty detector triggered → novel regime → activate safety protocol

---

## Technical Stack Recommendations

### Backend

**Primary: Python + FastAPI** — confirmed. This is the correct choice for an ML-heavy trading system.

Specific additions:
- **Celery + Redis** for task scheduling (pre-market scans, EOD reporting, weekend retraining)
- **WebSocket server** (via FastAPI) for real-time dashboard updates

### Database

**TimescaleDB (PostgreSQL extension)** for all time-series data:
- Trade logs with microsecond timestamps
- Feature snapshots at trade entry/exit
- Options chain snapshots (compressed, every 5 minutes during market hours)
- P&L time series

**Redis** for:
- Real-time Greeks cache
- Current prediction scores
- Session state (current regime, RCS, active positions)

**Why not InfluxDB/QuestDB:** TimescaleDB keeps everything in PostgreSQL, which means Supabase can host it natively. No additional infrastructure.

### ML Stack

- **PyTorch** for TFT (prediction engine)
- **LightGBM** for multi-day model and strategy scoring (faster than XGBoost for similar accuracy)
- **hmmlearn** for HMM regime detection
- **scikit-learn** for preprocessing, calibration, and evaluation
- **MLflow** for model versioning, experiment tracking, and deployment

### Real-Time Data

| Data Type | Recommended Provider | Cost Tier | Rationale |
|-----------|---------------------|-----------|-----------|
| Real-time SPX options chains | **CBOE LiveVol** or **Tradier** (already integrated) | $$ | Full chain with Greeks, Tradier gives execution + data |
| Historical options data | **ORATS** (5+ years) | $$ | Cleanest historical options data; academics and quant funds use it |
| VIX futures + term structure | **CBOE** direct feed | $ | Source of truth |
| Level 2 / order flow | **Polygon.io** or **Databento** | $$ | Polygon is cost-effective; Databento is higher quality |
| Economic calendar | **Quandl / Nasdaq Data Link** | $ | Structured, API-friendly |
| News sentiment | **Benzinga** or custom NLP on news APIs | $ | Pre-scored sentiment saves compute |

### Deployment

```
AWS Architecture:
├── ECS Fargate (FastAPI backend — auto-scaling)
├── Lambda (EOD reports, alerts)
├── RDS (TimescaleDB — Multi-AZ for durability)
├── ElastiCache (Redis)
├── S3 (model artifacts, trade logs archive)
├── SageMaker (model training — weekend batch jobs)
├── CloudWatch (monitoring, alerting)
└── API Gateway (WebSocket for dashboard)
```

**Why AWS over GCP/Azure:** Better ML tooling (SageMaker), more mature financial services infrastructure, better compliance posture for proprietary trading.

---

## Critical Risks & Blind Spots I See

### Risk 1: Latency Kills 0DTE

The brief doesn't specify latency requirements. For 0DTE, the difference between 500ms and 5-second execution can be thousands of dollars on a volatile day. **Requirement:** Order submission to Tradier must complete within 1 second. Prediction engine inference must complete within 2 seconds. Total decision-to-execution: <5 seconds.

### Risk 2: Tradier API Rate Limits & Reliability

The entire execution layer depends on a single broker API. If Tradier has an outage during a trading session with open positions, the system has no way to close positions. **Mitigation:** Add a secondary broker (Interactive Brokers API) as a failover for position closing only. Not for opening — just for emergency exits.

### Risk 3: Overfitting Dressed as "Learning"

The learning engine (Pillar 6) is the most dangerous component. A system that retrains too aggressively will overfit to recent conditions and blow up when conditions change. **Mitigation:** The "only deploy if Sharpe improves by >0.1" rule is critical. Also: track the number of retraining events — if the system is retraining every week, something is wrong.

### Risk 4: Data Quality Degradation

Options data feeds are notoriously messy — stale quotes, missing strikes, wrong Greeks calculations. A single bad data point (e.g., IV of 0%) can cascade through the prediction engine. **Mitigation:** Data quality checks before every prediction run: verify bid < ask, IV > 0, OI is non-negative, Greeks are within reasonable bounds.

### Risk 5: Human Override in V1 is a Bottleneck

V1 requires human approval for trades. If the human isn't available when the system identifies a high-conviction 0DTE setup at 10:15 AM, the opportunity is gone by 10:20 AM. **Mitigation:** Implement a tiered approval system: (a) pre-approved trade templates (e.g., "iron condor, <$X risk") execute automatically, (b) non-standard trades require approval, (c) all exits are automatic (never require human approval to close).

### Risk 6: The Brief Doesn't Address Slippage Modeling

Expected P&L calculations must include realistic slippage estimates, not theoretical mid-prices. SPX options typically have $0.05-0.20 slippage per leg; a 4-leg iron condor can lose $0.40-0.80 round-trip. On small premium trades ($2-3 credit), this is 15-30% of potential profit eaten by slippage.

---

## Open Questions Responses

### 1. Single most important thing to make this more profitable than any competitor?

**The dealer GEX model described in my highest-leverage proposal.** Most retail systems use lagging technical indicators. Most institutional systems have faster execution but less sophisticated signal generation for 0DTE. A real-time GEX model gives us the *why* behind intraday moves — dealer hedging flows are the largest deterministic force in SPX intraday price action. Combining GEX with the regime-gated capital allocation creates a system that knows both *what* will happen and *how much to bet*.

### 2. Single biggest risk for catastrophic failure?

**A liquidity vacuum during a position exit.** The -3% hard stop assumes we can close positions at reasonable prices. In a true flash crash, SPX option spreads blow out to $5-10 wide, and market orders fill at catastrophic levels. The hard stop could trigger a -3% *intended* loss that becomes -8% *actual* loss due to slippage. **Mitigation:** Pre-compute worst-case exit fills using current depth of book, not mid-prices. The true risk of a position is the loss at the worst realistic fill, not the theoretical max loss.

### 3. Data source most systems ignore?

**CBOE options open interest changes day-over-day, broken down by customer type.** The CBOE publishes data on whether new options positions are opened or closed, and whether they're customer or firm orders. This tells you whether the "smart money" (firms/dealers) is positioning directionally. Most systems look at volume; very few look at *who* is trading and *whether positions are opening or closing*.

### 4. System behavior during flash crash / circuit breaker?

**Immediate protocol:**
1. If SPX drops > 3% in < 15 minutes: cancel all open orders, market-close all positions (accept bad fills)
2. If Level 1 circuit breaker triggers (7% drop): system halts for the day — no reopening, no "buying the dip"
3. Post-event: flag the day as a structural break data point. Increase novelty detector sensitivity for 5 trading days.
4. Next day: reduce all position sizes by 75%, maximum 1 position (no satellites), for 3 trading days minimum

**Key principle:** In a flash crash, the priority is capital preservation, not profit capture. Every dollar saved is a dollar available to compound when normal conditions return.

### 5. Version 2 features?

In priority order:
1. **Full automation** — remove human approval requirement; system trades autonomously with SMS/push alerts
2. **Futures options integration** — /ES and /NQ options for overnight sessions and broader strategy universe
3. **Multi-account management** — run the system across multiple accounts with different risk profiles
4. **Earnings play module** — specialized model for trading options around SPX component earnings
5. **Portfolio hedging mode** — use the system to hedge an existing equity portfolio, not just speculate
6. **API for external signals** — allow injection of proprietary signals from other models/sources

---

> **Branch Status:** ✅ Complete — Ready for synthesis  
> **Confidence Level:** High across all pillars. Strongest conviction on Pillar 1 (GEX model) and Pillar 3 (risk management). Dashboard and learning engine proposals are solid but will need iteration during implementation.
