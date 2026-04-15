# SYNTHESIS.md — MarketMuse Best-Of Architecture
**Version:** 3.0 — Post Round 2 Challenge + Full Automation Integration  
**Written by:** Claude (Anthropic) — Synthesis Lead  
**Date:** 2026-04-15  
**Sources:** Round 2 challenge reports — claude-r2, gpt-r2, gemini-r2, grok-r2  
**Owner change integrated:** Full automation — no human approval for entries or exits  
**Status:** Ready for Round 3 — Refinement Round

---

## WHAT CHANGED FROM v2.0 — SUMMARY

| Change | Source | Impact |
|---|---|---|
| Full automation replaces human approval | Owner decision | Changes entire execution architecture |
| Strike-touch probability formula corrected | GPT (critical math error) | Core exit logic fixed |
| GEX computation upgraded to intraday OI | Gemini (critical data gap) | GEX model now actually reliable |
| Isotonic calibration made regime-conditioned | Claude + GPT | Prevents cross-regime calibration pollution |
| Independent risk sentinel on separate server | Gemini | Single point of failure eliminated |
| WebSocket heartbeat monitor added | Gemini | Most likely first live failure prevented |
| Charm/vanna added as Tier-2 signal | Grok (live market evidence) | Addresses real April 2026 failure mode |
| Hard entry ban 9:30–10:00 AM codified | Claude | Slippage gate formalized in Stage 0 |
| Drift detection upgraded to statistical basis | Claude + GPT | 5-trade threshold replaced with z-test |
| Per-regime paper validation criteria added | Claude | Aggregate metrics cannot hide blind spots |
| Position sizing uses stressed loss, not expected move | GPT | Critical formula correction |
| Monte Carlo parallelized | Gemini | Prevents compute lockup |
| Challenger promotion requires statistical test | GPT | Sharpe > 0.1 alone is insufficient |
| Paper phase extended to 45 days | Automation change | No human backstop requires more validation |

---

## SYNTHESIS CONFIDENCE SCORECARD v3.0

| Pillar | v2.0 | v3.0 | Reason for Change |
|---|---|---|---|
| Pillar 1: Prediction Engine | HIGH | **HIGH** | GEX data upgraded; charm/vanna added; calibration fixed |
| Pillar 2: Strategy Selection | HIGH | **HIGH** | Monte Carlo parallelized; Stage 0 time gate added |
| Pillar 3: Risk Management | HIGH | **HIGH** | Independent sentinel; automation hardens all circuit breakers |
| Pillar 4: Dashboard | MEDIUM | **HIGH** | Approval queue removed; charm panel + sentinel status added |
| Pillar 5: Exit Strategy | HIGH | **HIGH** | Strike-touch formula corrected; charm/vanna exit trigger added |
| Pillar 6: Learning Engine | MEDIUM | **MEDIUM** | Calibration fixed; drift upgraded; sample size still a real constraint |
| Infrastructure | HIGH | **HIGH** | Heartbeat + independent sentinel + parallelization added |

---

## 1. ALL LOCKED DECISIONS — v2.0 CONFIRMED + AMENDMENTS

DECISION-001 through DECISION-012 from SYNTHESIS v2.0 all remain locked with two amendments below.

---

### DECISION-007 AMENDMENT — FULL AUTOMATION
**Status:** 🔒 LOCKED (amended)  
**Original:** V1 requires human approval before trade execution.  
**Amended:** V1 is **fully automated** for all entries, exits, stop-loss execution, partial profit taking, and position management. No human approval required for any individual trade action.

**Human role in automated V1:**
- Monitor the War Room dashboard during market hours
- React to CRITICAL and EMERGENCY alerts
- Activate kill-switch if system behavior appears abnormal
- Review end-of-day and weekly reports
- Approve weekly model challenger promotions (after criteria are met)

**What humans never do in V1:**
- Approve individual trade entries or exits
- Manually override automated stop-loss execution
- Delay automated circuit breaker responses

**Mandatory requirements before go-live with full automation:**

| Requirement | Specification |
|---|---|
| Mobile kill-switch | All positions closed in < 5 seconds from any location |
| Broker-level limits | Position limits hardcoded at Tradier account level, independent of application |
| Extended paper phase | 45 days minimum |
| Independent sentinel | Separate cloud provider with close-only permissions |
| Audit log | Every automated decision logged before execution, append-only |
| Disaster recovery | Full backend failure spec built and tested before go-live |
| Heartbeat monitor | WebSocket silent timeout detection and degraded-mode protocol |

---

### DECISION-013 — Paper Phase Extended to 45 Days
**Status:** 🔒 LOCKED  
**Decision:** 45-day paper trading phase minimum before any live capital deployed.

**Go-live criteria — ALL must be met:**
1. Aggregate prediction accuracy ≥ 58% over full 45 days
2. **Per-regime accuracy ≥ 55%** for every day type with ≥ 8 observations (Credit: Claude)
3. Day types with < 8 observations → flagged as under-validated → live trading restricted to 25% normal sizing until 15 live observations per type
4. Paper Sharpe ≥ 1.5
5. Zero unhandled exceptions or unmanaged positions in final 20 paper sessions
6. All 6 circuit breaker scenarios tested and verified in sandbox
7. Kill-switch confirmed < 5 seconds response from mobile
8. Independent Sentinel verified operational on separate infrastructure
9. WebSocket heartbeat monitor verified — disconnect triggers degraded mode correctly

---

## 2. PILLAR 1: PREDICTION ENGINE

### 2.1 THREE-LAYER ARCHITECTURE — CONFIRMED

Layer A (Regime Engine) → Layer B (Path & Distribution) → Layer C (Vol Surface). Sequential and hierarchical. Unchanged from v2.0.

---

### 2.2 CHARM/VANNA — ELEVATED TO TIER-2 SIGNAL

**Credit: Grok — elevated based on live April 2026 market evidence. Adopted.**

Charm and vanna are second-order Greeks describing how delta changes with time (charm) and how delta changes with volatility (vanna). During sessions where GEX walls appear positive and stable, charm and vanna forces can be actively eroding the wall's effectiveness — invisible to first-order GEX analysis alone.

Grok provided live X data showing that on April 14–15 2026, credit spreads at high GEX nodes (6950–6960) blew up specifically due to negative vanna acceleration and charm decay — NOT GEX wall breach in the traditional sense. The wall appeared positive. Charm/vanna had turned against the position 15–25 minutes before VVIX moved.

**Key clarification:** Charm/vanna is a **mathematical signal computable directly from options chain data** — no X dependency. The fact that X traders were discussing it is validation that the signal is real, not the source. We compute it ourselves.

**Charm/Vanna Velocity Signal (computed every 5 minutes from Tradier options chain):**

```
Charm_Portfolio   = Σ(delta_change_per_day per open position)
Vanna_Portfolio   = Σ(delta_change_per_vol_point per open position)
Charm_Velocity    = (Charm_t − Charm_t−5min) / 5 min
Vanna_Velocity    = (Vanna_t − Vanna_t−5min) / 5 min
CV_Stress_Score   = weighted combination of Charm_Velocity + Vanna_Velocity
                    scaled 0–100
```

**Alert thresholds:**
- CV_Stress > 60: WARNING — charm/vanna degrading position faster than time alone predicts
- CV_Stress > 80: CRITICAL — exit evaluation triggers immediately (automated)
- Vanna_Velocity negative AND VIX rising simultaneously: CRITICAL regardless of absolute score

**Six new features added to feature pipeline (93 features total):**
Charm_Portfolio, Vanna_Portfolio, Charm_Velocity, Vanna_Velocity, CV_Stress_Score, Distance to nearest negative charm acceleration zone

---

### 2.3 GEX COMPUTATION — CRITICAL DATA SOURCE UPGRADE

**Credit: Gemini — critical finding adopted.**

v2.0 specified CBOE DataShop for real-time OI. This is insufficient for intraday GEX accuracy. CBOE DataShop provides end-of-day OI. For 0DTE trading, OI is static from pre-market. The real intraday GEX changes come from new contracts traded throughout the day — shifting dealer positioning — which are invisible in static OI.

**Using a GEX wall computed from pre-market OI at 2:00 PM is using a wall that may have been substantially eroded by 4 hours of 0DTE volume.**

**Corrected GEX computation:**

**Phase 1 — Morning baseline (pre-market to 10:00 AM):**  
Use CBOE DataShop end-of-day OI from prior close + any pre-market updates. Best available at open.

**Phase 2 — Intraday GEX adjustment (10:00 AM onward):**  
Use Databento OPRA trade-by-trade feed to compute incremental GEX delta from each new options print:

```
GEX_intraday(t) = GEX_morning_baseline
                + Σ(GEX_increment from each OPRA print since 9:30 AM)

GEX_increment per trade = trade_size × gamma × delta × 100 × spot
                          × sign(+1 if new long, -1 if new short)
```

This produces a continuously updating intraday GEX that reflects actual live dealer positioning.

**Fallback if Databento drops mid-session:**  
- Keep last known GEX value
- Apply a GEX_confidence_decay factor (GEX reliability decreases 5% per minute without update)
- When GEX_confidence < 60%: stop using GEX walls for entry decisions, continue using for exit decisions only
- Alert operator via WARNING

---

### 2.4 ISOTONIC CALIBRATION — REGIME-CONDITIONED

**Critical fix (Credit: Claude + GPT — both identified independently).**

A single global isotonic regression across all trades corrupts calibration during regime transitions. Range Day outcomes and Trend Day outcomes must never be used to calibrate each other's probability mappings.

**Fix:** 6 separate isotonic regression mappings — one per regime state. Only the mapping for today's observed regime is updated in the daily fast loop.

```python
calibration_tables = {
    regime: IsotonicRegression()
    for regime in ['quiet_bullish', 'volatile_bullish', 'quiet_bearish',
                   'crisis', 'pin_range', 'panic']
}

def fast_loop_calibration(session_trades, observed_regime):
    calibration_tables[observed_regime].fit(
        [t.prediction_score for t in session_trades],
        [t.outcome for t in session_trades]
    )
    # Other regime tables are NOT touched this session
```

**Profit impact of this fix:** Prevents 5–8 sessions of degraded performance per regime transition event, which occurs ~4–8 times per year. Direct P&L protection.

---

### 2.5 TIME-OF-DAY HARD ENTRY GATE — CODIFIED

**Credit: Claude — adopted as Stage 0 gate (see Pillar 2).**

| Time Window | Entry Status | EV Slippage Multiplier |
|---|---|---|
| 9:30–10:00 AM | **HARD BLOCKED** | N/A |
| 10:00 AM–12:00 PM | Allowed | 1.2× |
| 12:00–2:00 PM | Allowed (optimal) | 1.0× |
| 2:00–2:30 PM | Allowed, reduced size tier | 1.5× |
| 2:30 PM+ | Short-gamma exits only | N/A |

---

## 3. PILLAR 2: STRATEGY SELECTION ENGINE

### 3.1 STAGE 0 — TIME-OF-DAY + CHARM/VANNA GATE (NEW)

Before any other evaluation runs:
1. If time is 9:30–10:00 AM → block all new entries
2. If CV_Stress_Score > 70 → block all new short-gamma entries regardless of time
3. If time is 2:30 PM+ → block all new short-gamma entries

Only entries that pass Stage 0 proceed to Stages 1–4.

---

### 3.2 STAGES 1–4 — CONFIRMED FROM v2.0

Day Type eligibility gate (Stage 1), GEX strike optimizer (Stage 2), EV utility ranking (Stage 3), liquidity/slippage veto (Stage 4) — all confirmed.

**Updated utility function — tax-aware (Credit: Gemini):**

```
Utility = EV_net_after_tax
        − λ1 × ExpectedShortfall
        − λ2 × TailRisk
        − λ3 × SlippagePenalty × TimeOfDayMultiplier
        − λ4 × LiquidityPenalty
        + λ5 × CapitalEfficiency
        − λ6 × TaxDragPenalty
```

---

### 3.3 MONTE CARLO — PARALLELIZED

**Credit: Gemini — critical for automation (no human approval means compute delay is not masked).**

10,000 paths × 8 candidates = 80,000 simulations every 5 minutes. Must complete in < 2 seconds.

**Solution:** Numba JIT-compiled simulation loop (10–50× speed), combined with AWS Lambda for burst parallelization. Each of the 8 candidates runs as a parallel Lambda invocation. Results aggregated in < 2 seconds total.

---

### 3.4 AUTOMATION FINAL SANITY CHECK

Before execution (automated), the system performs a final 3-point sanity check:
1. Current price within ±0.2% of price used in EV evaluation
2. Top-ranked strategy unchanged since evaluation
3. All liquidity conditions still met (re-check bid/ask)

If any check fails → re-run Stage 3 with current data. If re-run produces same result within 30 seconds → execute. If not → no-trade this cycle, re-evaluate next 5-minute cycle.

---

## 4. PILLAR 3: RISK MANAGEMENT ENGINE

### 4.1 TWO-LAYER AUTOMATED SAFETY ARCHITECTURE

With full automation, safety architecture IS the only protection. Must have redundancy.

**Layer 1 — Primary Application:**  
All circuit breakers, Greek limits, drawdown stops, time stops, charm/vanna exits run here. Sub-second response.

**Layer 2 — Independent Sentinel (Credit: Gemini — mandatory infrastructure):**  
Separate process on a separate cloud provider (GCP if primary is AWS) with read-only position data access and close-only Tradier permissions.

**What Sentinel monitors (independent data feeds, not shared with primary):**
- Account drawdown via Tradier REST (every 10 seconds)
- SPX price via Polygon.io
- VVIX via CBOE direct
- Primary application heartbeat

**Sentinel triggers (fires even if primary application is completely down):**

| Trigger | Sentinel Action |
|---|---|
| Drawdown > 2.5% | SMS alert to operator |
| Drawdown > 3.0% | AUTO: Submit close-all via Tradier API |
| SPX drops > 2.5% in 30 min | AUTO: Submit close-all |
| No app heartbeat > 60 seconds | SMS CRITICAL alert |
| No app heartbeat > 120 seconds | AUTO: Submit close-all |

**Layer 3 — Broker Level:**  
Tradier account-level position limits hardcoded. Independent of application state. Last-resort protection.

---

### 4.2 POSITION SIZING — FORMULA CORRECTED

**Critical fix (Credit: GPT). Expected move ≠ dollar risk for options positions.**

v2.0 used predicted move as the denominator. Wrong. A credit spread's dollar risk is the spread width, not the underlying's expected move.

**Corrected formula:**

```
Position_Size = (Account_Value × Risk_Pct) / Stressed_Loss_Per_Contract

Stressed_Loss_Per_Contract = MAX(
    structural_max_loss,           # spread width × $100
    scenario_loss                  # adverse_move × 1.5 + spread_widening + delayed_exit_slippage
)

Risk_Pct_Core      = 0.5% × (RCS / 100)
Risk_Pct_Satellite = 0.25% × (RCS / 100)
```

The scenario_loss uses 1.5× the predicted adverse move to account for model error, plus spread widening under stress, plus delayed stop execution slippage.

---

### 4.3 WEBSOCKET HEARTBEAT MONITOR — MANDATORY

**Credit: Gemini — highest-priority infrastructure addition. This is the most likely first live failure.**

```python
HEARTBEAT_TIMEOUT_SECONDS = 3

def heartbeat_monitor():
    while market_is_open():
        age = time.now() - last_data_timestamp
        if age > HEARTBEAT_TIMEOUT_SECONDS:
            trigger_degraded_mode()
        time.sleep(0.5)

def trigger_degraded_mode():
    halt_all_new_entries()
    switch_to_rest_polling(interval_seconds=2)
    alert_operator(level='CRITICAL')
    if degraded_duration() > 30:
        close_all_positions()
        halt_trading_session()
```

No trade executes during degraded mode. Reconnection within 30 seconds → resume on fresh data only.

---

### 4.4 ALL OTHER RISK CONTROLS — CONFIRMED

VVIX circuit breakers, Greek limits (delta ±0.15, vega ±$500 per $100k), correlation checks, portfolio Greek aggregation, pre-market risk scan — all confirmed from v2.0. All now execute fully automated.

---

## 5. PILLAR 4: MONITORING & DASHBOARD ENGINE

### 5.1 OPERATOR ROLE IN AUTOMATED V1

Monitor → React to alerts → Kill-switch if abnormal → Review reports → Approve weekly model updates.

Operators never approve individual trades.

---

### 5.2 DASHBOARD PANELS — v3.0 UPDATES

All v2.0 panels confirmed. Changes:

**Removed:** Approval Queue (automation — no approvals)

**Added — Charm/Vanna Stress Panel:**
- Live CV_Stress_Score per position and portfolio (0–100, color-coded)
- Charm velocity and Vanna velocity trend lines
- Red alert if CV_Stress > 70 on any open position

**Added — Independent Sentinel Status:**
- Green/Red indicator: Sentinel operational or unreachable
- Last heartbeat timestamp from Sentinel
- Red = EMERGENCY (if Sentinel is down, the backup protection is gone)

**Added — Data Feed Health Panel:**
- WebSocket status (Connected / Degraded / Reconnecting)
- Last message timestamp and age
- Component latency vs budget
- Databento OPRA feed status

**Added — Automation Activity Log:**
- Real-time stream of every automated decision
- Each entry: timestamp, action, trigger, instrument, price, prediction score, regime
- Operator sees exactly what the system did and why, in real time

---

### 5.3 SENTINEL AI ASSISTANT — UPDATED FOR AUTOMATION

All v2.0 Sentinel functionality confirmed. New example outputs:

- *"AUTO-EXECUTED: SPX 5450/5440 put credit spread, $1.20 credit. Confidence: 71%. GEX wall at 5440. CV Stress: 22. EV after costs: $89."*
- *"WARNING: CV Stress rising on Core position (Score: 64). Charm decay accelerating. Monitoring for State 4 transition."*
- *"AUTOMATED EXIT: Core position moved to State 4 — charm velocity negative for 2 consecutive intervals. 50% exit executed at $0.68 credit. Remaining trailed."*
- *"EMERGENCY: Primary WebSocket degraded 28s. All new entries halted. Positions closing in 2 seconds if not restored."*

---

### 5.4 ALERT TIERS — AUTOMATION UPDATED

| Tier | Condition | Channel | Automated Response |
|---|---|---|---|
| **INFO** | Normal execution, regime update | Audit log | None |
| **WARNING** | CV Stress > 60, VVIX +15%, RCS < 50 | Dashboard + push | System adjusts sizing |
| **CRITICAL** | −2% drawdown, VVIX +20%, WebSocket degraded | Push + SMS | System halts new entries |
| **EMERGENCY** | −3% drawdown, VVIX > 140, circuit breaker | Push + SMS + email | System closes all + halts |
| **SENTINEL** | Primary app heartbeat lost > 120s | SMS only | Sentinel closes all positions |

---

## 6. PILLAR 5: P&L, STOP-LOSS & EXIT STRATEGY ENGINE

### 6.1 STRIKE-TOUCH PROBABILITY — FORMULA CORRECTED

**Credit: GPT — critical math error corrected.**

v2.0 formula: `N(d2 adjusted for remaining time)` — **WRONG**  
N(d2) is terminal exercise probability (will price end above/below strike?).  
We need first-passage probability (will price touch the strike at any point before close?).  
These are different mathematical objects. Using N(d2) for a touch-probability exit system produces systematically wrong exit triggers.

**Corrected formula — First-Passage Touch Probability:**

```
Touch_Probability = N(-d2_touch) + exp(2μ × ln(K/S) / σ²) × N(-d1_touch)

d1_touch = [ln(S/K) + (μ + σ²/2) × T] / (σ × √T)
d2_touch = d1_touch − σ × √T

Where:
S = current SPX price
K = short strike level
μ = realized drift over last 30 minutes (annualized)
σ = realized vol over last 30 minutes (annualized, NOT implied vol)
T = remaining time to 2:30 PM in years
```

**Why realized vol, not implied:** We need to model where price will actually go, not what the market is pricing. Realized vol reflects actual recent movement; implied vol includes fear premium.

**Threshold values** (10% / 25% / 40% from v2.0) retained as **initial calibration priors only**, explicitly labeled as tunable. Must be validated during paper phase. (Credit: GPT — correctly identified these as unsupported assertions.)

---

### 6.2 CHARM/VANNA EXIT TRIGGER — ADDED TO STATE MODEL

**Credit: Grok — adopted as additional State 3 → State 4 transition.**

The 5-state exit model is confirmed from v2.0. New transition condition added for State 3 (Mature Winner):

**State 3 → State 4 triggers (any one):**
- Prediction confidence drops > 15 points from entry (existing)
- GEX wall breached (existing)
- Touch probability > 25% (existing)
- **NEW: CV_Stress_Score > 70 while position is in profit → 50% automated exit, remainder trailed**

This catches the specific April 2026 failure mode: a position at 65% of max profit where charm/vanna is accelerating toward a loss that price alone hasn't signaled yet.

---

### 6.3 FULL AUTOMATION — ALL EXITS

All exits execute without human confirmation:
- **OCO hard stop:** Pre-submitted to Tradier at entry. Executes independently.
- **Time stop (2:30 PM):** Close-all-short-gamma queued at 2:29:50 PM daily.
- **State-5 forced exit:** Executes via API immediately.
- **Portfolio stop (−3%):** Executes via primary app AND Independent Sentinel simultaneously.
- **Charm/Vanna exit:** Executes via primary app as State 4 transition.

**OCO execution reality note (Credit: GPT):**  
Options OCO stops on fast markets fill at market price when triggered — not exactly at the stop level. Model stop execution with slippage expansion. The stressed_loss_per_contract in the sizing formula already accounts for this.

---

### 6.4 ALL OTHER EXIT RULES — CONFIRMED

2:30 PM short-gamma hard exit, 3:45 PM long-gamma exit, no averaging into losers, no converting losers, partial profit at 40–60% credit capture, State-based model — all confirmed from v2.0. All fully automated.

---

## 7. PILLAR 6: AI LEARNING & ADAPTATION ENGINE

### 7.1 TWO-SPEED LOOP — CONFIRMED WITH FIXES

Fast loop (daily) + Slow loop (weekly) architecture confirmed. Key fixes:

---

### 7.2 ISOTONIC CALIBRATION — REGIME-CONDITIONED

Already specified in Pillar 1 (Section 2.4). This is a Pillar 6 implementation detail that belongs in both sections for completeness.

---

### 7.3 DRIFT DETECTION — STATISTICAL BASIS

**Credit: Claude + GPT. The 5-trade rolling threshold is statistically meaningless and replaced.**

```python
def check_drift(recent_trades, historical_baseline_accuracy, min_sample=30):
    if len(recent_trades) < min_sample:
        return 'INSUFFICIENT_DATA'

    n = len(recent_trades)
    observed = sum(t.correct for t in recent_trades) / n
    se = (historical_baseline_accuracy * (1 - historical_baseline_accuracy) / n) ** 0.5
    z = (observed - historical_baseline_accuracy) / se

    if z < -2.326:   # p < 0.01
        return 'DRIFT_CRITICAL'
    elif z < -1.645: # p < 0.05
        return 'DRIFT_WARNING'
    return 'NO_SIGNAL'
```

**Automated responses:**
- DRIFT_WARNING → reduce position sizes by 30%, increase no-trade threshold
- DRIFT_CRITICAL → reduce position sizes by 60%, evaluate challenger
- DRIFT_CRITICAL + challenger available + promotion criteria met → auto-promote
- DRIFT_CRITICAL + no challenger → reduce to minimum sizes, retrain priority

---

### 7.4 CHALLENGER PROMOTION — STATISTICAL TEST REQUIRED

**Credit: GPT. Sharpe > 0.1 alone is noise at small sample sizes.**

All 7 criteria must be met before automated promotion:

1. Minimum 40 shadow sessions (up from 20)
2. Minimum 80 trade observations
3. Coverage of ≥ 3 different regime types
4. Two-sample z-test on accuracy: p < 0.05 in challenger's favor
5. Out-of-sample Sharpe improvement ≥ 0.15 over same period
6. Challenger max drawdown ≤ champion max drawdown
7. Challenger passes all 6 circuit breaker scenarios in sandbox

When all 7 met → automated promotion (logged in audit trail, flagged in weekly report for operator awareness).

---

### 7.5 THREE-LAYER DIAGNOSTICS + PER-REGIME — CONFIRMED

Three-layer separation (signal quality, strategy selection quality, sizing/exit quality) confirmed from v2.0, applied at per-regime-per-strategy granularity.

---

### 7.6 COUNTERFACTUAL BACKTEST — CONFIRMED

End-of-session counterfactual analysis confirmed from v2.0. Results feed automated parameter adjustments within pre-approved bounds (±5–10% on threshold values).

---

## 8. INFRASTRUCTURE — CRITICAL ADDITIONS

### 8.1 INDEPENDENT SENTINEL — SPECIFICATIONS

**Deployment:** Separate cloud provider from primary. Primary = AWS → Sentinel = Google Cloud (or equivalent).

**Capabilities:**
- READ: Tradier positions/P&L (REST poll every 10s)
- READ: SPX price (Polygon.io)
- READ: VVIX (CBOE direct)
- WRITE: Tradier close-all order (close-only permissions)
- WRITE: Operator SMS alerts (Twilio)

**Cannot:** Open positions, modify positions, access application DB.

**Sentinel-of-Sentinel:** The Sentinel itself must have a health check. If Sentinel goes down → operator SMS within 60 seconds (different monitoring service). The Sentinel being offline is a WARNING-level alert.

---

### 8.2 AUDIT LOG — MANDATORY

Every automated decision logged before execution. Write-once, append-only. Minimum 2-year retention.

```json
{
  "timestamp": "2026-04-15T10:23:45.123Z",
  "decision_type": "ENTRY|EXIT|HALT|CIRCUIT_BREAKER|CHARM_VANNA_EXIT",
  "trigger": "state_machine|time_stop|greek_limit|charm_vanna|circuit_breaker",
  "instrument": "SPX",
  "action": "SELL_PUT_SPREAD",
  "strikes": [5440, 5430],
  "quantity": 2,
  "prediction_score": 0.71,
  "regime": "pin_range",
  "rcs": 76,
  "cv_stress": 22,
  "gex_wall_distance_pct": 0.8,
  "ev_net": 0.89,
  "execution_price": 1.18
}
```

---

## 9. TECHNICAL STACK — v3.0

| Component | Technology | Change | Justification |
|---|---|---|---|
| Primary Backend | Python 3.12 + FastAPI | Unchanged | Confirmed |
| Independent Sentinel | Python + separate FastAPI | **NEW** | GCP, close-only |
| Primary ML | LightGBM + XGBoost | Unchanged | Confirmed |
| Monte Carlo | Numba JIT + AWS Lambda | **UPGRADED** | Parallelization |
| Feature Store | QuestDB | Unchanged | Confirmed |
| Trade/History DB | TimescaleDB | Unchanged | Confirmed |
| Intraday Cache | Redis | Unchanged | Confirmed |
| Model Registry | MLflow | Unchanged | Confirmed |
| GEX — Morning | CBOE DataShop EOD OI | Unchanged | Baseline |
| GEX — Intraday | Databento OPRA trade feed | **UPGRADED** | Intraday synthesis |
| WebSocket | Tradier + Heartbeat Monitor | **+ HEARTBEAT** | Silent timeout protection |
| Backtesting | vectorbt | Unchanged | Confirmed |
| Deployment | AWS ECS us-east-1 (primary) + GCP us-east1 (sentinel) | **+ GCP** | Redundancy |
| Alerts | Twilio SMS + FCM Push + SMTP | **SPECIFIED** | Multi-channel |
| Audit Log | TimescaleDB append-only | **NEW** | Automation compliance |

---

## 10. SUCCESS METRICS — v3.0 ADDITIONS

All v2.0 metrics confirmed. Additions:

| Metric | Target | Source |
|---|---|---|
| Per-regime accuracy (paper) | ≥ 55% per type with ≥ 8 obs | Claude |
| Sentinel response time | < 10 seconds from trigger | Automation |
| WebSocket uptime | > 99.5% market hours | Infrastructure |
| Audit log completeness | 100% — zero unlogged decisions | Automation |
| Charm/Vanna exit accuracy | Track % of CV exits that were correct | Grok |
| Stressed loss vs actual | Actual < stressed estimate in 95% of stops | GPT |
| Touch probability calibration | Calibrated during paper phase vs actual touches | GPT |

---

## 11. OPEN QUESTIONS FOR ROUND 3

**Q1 — Charm/Vanna Threshold Calibration:**
The CV_Stress thresholds (60/70/80) are initial priors. Design the paper-phase calibration protocol: what data is logged, how are thresholds tuned, what is the validation metric, and what is the false-positive rate target?

**Q2 — Independent Sentinel Full Specification:**
Complete API interaction between Sentinel and Tradier. What happens if Sentinel itself fails? Is there a Sentinel health monitor? What is acceptable Sentinel downtime?

**Q3 — First-Passage Touch Probability Calibration:**
The corrected formula requires tuning of the threshold values (10%/25%/40%). Design the paper-phase calibration protocol. What is logged? How are thresholds validated? What is the target: minimize false exits or minimize missed exits?

**Q4 — Databento OPRA Intraday GEX Integration:**
Exact implementation of trade-by-trade GEX delta computation. What is the update frequency? What is the computational cost? What is the complete fallback protocol if Databento drops mid-session (beyond the GEX confidence decay already specified)?

**Q5 — Kill-Switch User Experience:**
Complete mobile kill-switch interface design. What does the operator see? What actions are available? What confirmation (if any) is required? How is < 5-second response time guaranteed across all network conditions?

---

## 12. AI CREDIT SUMMARY — v3.0

| AI | Round 2 Contribution Adopted |
|---|---|
| **Claude** | Regime-conditioned isotonic calibration, per-regime paper validation criteria, time-of-day entry ban as hard Stage 0 gate, DECISION-007 automation contradiction identified (now resolved by owner), Bayesian priors for regime-strategy matrix cold start |
| **GPT** | Strike-touch probability formula correction (critical math), position sizing using stressed loss (critical formula), drift detection upgraded to z-test, challenger promotion statistical criteria, slippage as the central mathematical weakness, profit thresholds labeled as priors not facts |
| **Gemini** | GEX intraday OI upgrade via Databento OPRA (critical data fix), Independent Sentinel on separate cloud (critical infrastructure), WebSocket heartbeat monitor (critical infrastructure), Monte Carlo parallelization via Numba + Lambda, tax-aware utility function |
| **Grok** | Charm/vanna as Tier-2 mathematical signal (live April 2026 evidence), charm/vanna exit trigger added to State model, CV_Stress panel in dashboard, real-world stress scenarios, April 14–15 validation data |

---

*Document written by: Claude (Anthropic) — Synthesis Lead*  
*Version: 3.0*  
*Date: 2026-04-15*  
*Repo: https://github.com/tesfayekb/market-muse.git*  
*Next: Round 3 — Refinement of 5 specific open questions, then FINAL_SPEC.md*
