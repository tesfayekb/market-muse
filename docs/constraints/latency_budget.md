# latency_budget.md — System Latency Requirements
**Repo:** https://github.com/tesfayekb/market-muse.git  
**Path:** `/docs/constraints/latency_budget.md`  
**Purpose:** Defines maximum acceptable latency for every system component. Latency is a direct drag on 0DTE profitability.

---

## ⚠️ RULE FOR ALL AI CONTRIBUTORS

Every architecture proposal must respect these latency budgets. If your proposed component cannot meet its budget, you must either simplify the component or propose a caching/pre-computation strategy that achieves the budget.

---

## THE FUNDAMENTAL RULE

> **On 0DTE trading, time IS money.**  
> SPX can move 5 points in 10 seconds during active trading.  
> Every second of latency between signal and fill is potential profit left on the table — or a loss that wasn't stopped in time.

---

## LATENCY BUDGET BY COMPONENT

### Critical Path (Signal → Fill)

This is the most important latency chain in the entire system.

```
Market Data Update
       ↓  [<50ms]
Prediction Engine Update
       ↓  [<500ms]
Strategy Selection Decision
       ↓  [<200ms]
Risk Check (position size, Greeks limits)
       ↓  [<100ms]
Order Construction
       ↓  [<100ms]
API Submission to Tradier
       ↓  [<300ms]
Fill Confirmation
       ↓  [<500ms]
Position State Update
       ↓  [<100ms]
Dashboard Update

TOTAL CRITICAL PATH TARGET: <1.5 seconds
TOTAL CRITICAL PATH MAXIMUM: <5 seconds
```

---

### Component-Level Budgets

| Component | Target | Maximum | Notes |
|---|---|---|---|
| Market data ingestion | <50ms | <200ms | Streaming WebSocket |
| Feature engineering (signal calculation) | <200ms | <500ms | Must be pre-computed where possible |
| Prediction engine inference | <300ms | <1s | Use cached models, not real-time training |
| Regime detection update | <500ms | <2s | Updates every 60 seconds, not per tick |
| Strategy selection | <200ms | <500ms | Rule-based + ML scoring |
| Risk check (position level) | <50ms | <200ms | Must be near-instant |
| Risk check (portfolio level) | <100ms | <300ms | Aggregated Greeks calculation |
| Order construction | <100ms | <200ms | |
| Tradier API submission | <300ms | <1s | Network dependent |
| Fill confirmation | <500ms | <2s | Exchange dependent |
| Stop/target order submission (OCO) | <300ms | <1s | Immediately after fill |
| Position state update | <100ms | <200ms | |
| Dashboard refresh | <1s | <3s | Must NOT block trading thread |
| Alert dispatch (push/email) | <2s | <10s | Non-critical path |

---

## NON-CRITICAL PATH BUDGETS

These run on background threads and must not interfere with the critical path:

| Component | Target | Frequency |
|---|---|---|
| Options chain full refresh | <2s | Every 60 seconds |
| Greeks recalculation (portfolio) | <500ms | Every 30 seconds |
| Account/margin update | <1s | Every 60 seconds |
| Model drift detection | <5s | Every 5 minutes |
| Trade logging to database | <500ms | Per trade event |
| Performance metrics update | <2s | Every 5 minutes |
| Regime confidence score recalculation | <2s | Every 5 minutes |

---

## INFRASTRUCTURE REQUIREMENTS TO MEET THESE BUDGETS

### Minimum Cloud Specifications

| Component | Minimum Spec | Recommended Spec |
|---|---|---|
| Prediction engine server | 4 vCPU, 16GB RAM | 8 vCPU, 32GB RAM |
| API/backend server | 2 vCPU, 8GB RAM | 4 vCPU, 16GB RAM |
| Database | 2 vCPU, 8GB RAM, SSD | 4 vCPU, 16GB RAM, NVMe SSD |
| Message queue | Redis or equivalent | Redis Cluster |

### Cloud Region
- Must be deployed in **US-East (us-east-1 or equivalent)** — closest to NYSE/CBOE
- Cross-region latency adds 20–80ms — unacceptable for 0DTE critical path

### Architecture Requirements
- Prediction engine must load models into memory at session start — no disk reads during trading hours
- Feature calculations must use in-memory data structures, not database queries, during the critical path
- Dashboard must run on a separate thread/process from the trading engine — never block trading for UI updates

---

## LATENCY MONITORING REQUIREMENT

The system must log and alert on latency violations:

| Violation Level | Condition | Action |
|---|---|---|
| Warning | Any component exceeds target by 2x | Log + dashboard indicator |
| Critical | Critical path total exceeds 3 seconds | Log + alert + investigate |
| Emergency | Critical path total exceeds 10 seconds | Log + alert + pause new entries |

---

*Maintained by: tesfayekb | Repo: https://github.com/tesfayekb/market-muse.git*
