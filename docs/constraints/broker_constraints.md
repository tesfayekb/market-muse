# broker_constraints.md — Tradier API Constraints & Limits
**Repo:** https://github.com/tesfayekb/market-muse.git  
**Path:** `/docs/constraints/broker_constraints.md`  
**Purpose:** Every AI proposal must be compatible with these hard constraints. Ignoring them produces architectures that fail in live execution.

---

## ⚠️ RULE FOR ALL AI CONTRIBUTORS

Before proposing ANY execution, order management, or real-time data logic — read this file completely. Proposals that violate these constraints will be rejected in synthesis.

---

## 1. TRADIER API — CORE FACTS

| Property | Detail |
|---|---|
| **Broker** | Tradier Brokerage |
| **API Type** | REST + Streaming (WebSocket) |
| **Base URL** | `https://api.tradier.com/v1/` |
| **Auth** | OAuth 2.0 Bearer Token |
| **Account Type Required** | Margin account for options selling |
| **Supported Instruments** | Equities, ETFs, Options (US markets) |
| **SPX/NDX/RUT Support** | ✅ Yes — index options supported |
| **0DTE Support** | ✅ Yes |
| **Paper Trading** | ✅ Sandbox environment available |

---

## 2. API RATE LIMITS

| Endpoint Category | Rate Limit |
|---|---|
| **Market Data (quotes)** | 60 requests/minute |
| **Options Chain** | 60 requests/minute |
| **Order Placement** | 60 requests/minute |
| **Account/Positions** | 60 requests/minute |
| **Streaming (WebSocket)** | 1 persistent connection per session |

**Critical Implication for System Design:**
- You CANNOT poll the full SPX options chain every second via REST — use streaming where possible
- Options chain polling at 60 req/min means roughly 1 full chain refresh per second maximum — and that uses your entire quota
- **Architecture must be streaming-first, REST for snapshots and order management only**

---

## 3. ORDER TYPES SUPPORTED

| Order Type | Supported | Notes |
|---|---|---|
| Market Order | ✅ | Use with caution on options — wide spreads |
| Limit Order | ✅ | Preferred for all options entries and exits |
| Stop Order | ✅ | Price-based stop |
| Stop Limit | ✅ | Stop + limit combo |
| Trailing Stop | ✅ | Percentage or dollar-based |
| OCO (One Cancels Other) | ✅ | Critical for simultaneous SL + TP orders |
| Multi-leg (Spreads) | ✅ | Supported — use for iron condors, verticals, etc. |
| Bracket Order | ✅ | Entry + SL + TP in one order |

**Critical for Strategy Design:**
- OCO orders are available — use them. Every trade should have SL and TP submitted as OCO immediately on fill
- Multi-leg orders reduce execution risk on spreads — always use multi-leg for spreads, never leg in separately on V1
- **Never use market orders for SPX options** — bid/ask spreads can be $0.50–$2.00 wide on low-liquidity strikes. Always limit orders.

---

## 4. REAL-TIME DATA VIA TRADIER

### What Tradier Provides
| Data Type | Available | Method |
|---|---|---|
| Real-time quotes (equities) | ✅ | Streaming |
| Real-time options quotes | ✅ | Streaming + REST |
| Options chain (full) | ✅ | REST (snapshot) |
| Greeks (Delta, Gamma, Theta, Vega) | ✅ | Included in options chain |
| IV (Implied Volatility) | ✅ | Per-strike in options chain |
| Historical OHLCV | ✅ | REST |
| Level 2 quotes | ❌ | Not available via Tradier |
| Time & Sales | ❌ | Not available via Tradier |
| Dark pool data | ❌ | Not available via Tradier |

**Critical Gaps — Must Source Externally:**
- Level 2 order book data → requires separate data provider (recommend: CBOE DataShop, Interactive Brokers data feed, or Polygon.io)
- Dark pool / unusual options activity → requires separate provider (recommend: Unusual Whales, Flow Algo, or Market Chameleon)
- VIX futures curve → requires separate source (CBOE direct or Bloomberg)

---

## 5. EXECUTION LATENCY BUDGET

For 0DTE trading, latency directly impacts profitability. These are realistic targets:

| Component | Target Latency | Max Acceptable |
|---|---|---|
| Signal generation (prediction) | <500ms | <2s |
| Strategy selection decision | <200ms | <500ms |
| Order construction | <100ms | <200ms |
| API order submission | <300ms | <1s |
| Fill confirmation receipt | <500ms | <2s |
| **Total round-trip (signal → fill)** | **<1.5s** | **<5s** |

**Why This Matters:**
- SPX 0DTE moves fast. A 10-point move on SPX takes ~30–60 seconds in a trending environment
- If your signal-to-fill latency is 10 seconds, you're entering after the move has already begun
- Every 1 second of unnecessary latency is a direct P&L drag

**Architecture Implications:**
- Prediction engine must run on low-latency infrastructure (co-located or cloud with <50ms to NYSE)
- No synchronous blocking calls in the execution path
- Dashboard updates must NOT block the trading execution thread

---

## 6. OPTIONS-SPECIFIC EXECUTION CONSTRAINTS

### Bid/Ask Spread Reality (SPX 0DTE)
| Time of Day | Typical Spread (ATM) | Typical Spread (OTM) |
|---|---|---|
| Market open (9:30–10:00) | $1.00–$3.00 | $0.50–$2.00 |
| Mid-morning (10:00–12:00) | $0.25–$0.75 | $0.20–$0.50 |
| Midday (12:00–2:00) | $0.20–$0.50 | $0.15–$0.40 |
| Power hour (3:00–4:00) | $0.50–$2.00 | $0.30–$1.50 |
| Last 15 min (3:45–4:00) | $2.00–$10.00+ | Wide/illiquid |

**Critical Rule:** Do NOT trade in the first 30 minutes unless the system has very high conviction. Opening spreads are 3–5x wider — this is pure slippage expense.

**Critical Rule:** Do NOT hold 0DTE positions past 3:45 PM EST — enforced by hard stop in system. After 3:45, spreads explode and liquidity disappears.

### Strike Selection Constraints
- Always check open interest > 500 contracts before entering a strike
- Always check daily volume > 100 contracts before entering
- Prefer strikes within 1–2% of ATM for highest liquidity
- Deep OTM strikes (<0.10 delta) have poor fill quality — avoid in V1

---

## 7. MARGIN & BUYING POWER CONSTRAINTS

| Strategy | Margin Requirement |
|---|---|
| Long call/put | Full premium (defined risk) |
| Vertical spread (debit) | Full debit paid |
| Vertical spread (credit) | Width of spread minus credit received |
| Iron condor | Width of wider spread minus total credit |
| Iron butterfly | Width of wings minus total credit |
| Naked short (undefined risk) | ❌ NOT PERMITTED in V1 |

**Hard Rules:**
- No undefined-risk positions in V1 (no naked calls, no naked puts, no ratio spreads with undefined side)
- Maximum margin utilization: 70% of available buying power (30% always held as buffer)
- If margin utilization exceeds 80% intraday, the system flags for review before any new entries

---

## 8. ORDER MANAGEMENT CONSTRAINTS

| Rule | Detail |
|---|---|
| Max open orders | No hard Tradier limit, but system enforces max 3 positions (1 core + 2 satellites) |
| Order modification | Supported — cancel and replace |
| Partial fills | Possible — system must handle partial fill state |
| Order rejection | Must be handled gracefully — retry logic required |
| GTC orders | Supported — but 0DTE positions should be DAY orders only |

---

## 9. WHAT TO BUILD AROUND THESE CONSTRAINTS

**For Prediction Engine:**
- Source options flow data externally (Tradier doesn't provide unusual activity alerts)
- Use streaming for real-time quote monitoring, REST for periodic chain snapshots

**For Risk Management:**
- Build margin utilization monitor using Tradier account endpoint (poll every 30 seconds)
- Greeks come from Tradier options chain — recalculate portfolio-level Greeks after every position change

**For Execution Engine:**
- Always use limit orders for options
- Always submit OCO (SL + TP) immediately after fill confirmation
- Never use multi-leg orders with more than 4 legs in V1 (complexity vs fill quality tradeoff)
- Build retry logic for order rejections — Tradier occasionally rejects during high-volume market events

**For Dashboard:**
- Tradier provides real-time position P&L — use streaming endpoint, not polling
- Account balance updates available via REST — poll every 60 seconds is sufficient

---

*Maintained by: tesfayekb | Repo: https://github.com/tesfayekb/market-muse.git*
