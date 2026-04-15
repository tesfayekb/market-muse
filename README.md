# 🧠 MarketMuse — AI-Powered Options Trading System
**Repository:** https://github.com/tesfayekb/market-muse.git  
**Mission:** Build the most profitable options trading system in the world.  
**Status:** Phase 1 — Multi-AI Brainstorm & Architecture Design

---

## ⚡ ONE RULE ABOVE ALL OTHERS

> **PROFIT. PROFIT. PROFIT.**  
> Every idea, every architecture decision, every feature, every line of code must be justified by one question:  
> **"Does this make the system MORE profitable?"**  
> If the answer is not a clear YES — it does not belong in this system.

---

## 📁 Repository Structure

```
/
├── README.md                        ← You are here — start here
├── AI_INSTRUCTIONS.md               ← Universal rules ALL AIs must follow
├── MASTER_BRIEF.md                  ← Full system specification (read before anything else)
│
├── /docs
│   ├── /ai-branches
│   │   ├── claude-branch.md         ← Claude (Anthropic) proposals
│   │   ├── gpt-branch.md            ← GPT-4o (OpenAI) proposals
│   │   ├── gemini-branch.md         ← Gemini (Google) proposals
│   │   └── grok-branch.md           ← Grok (xAI) proposals
│   │
│   ├── SYNTHESIS.md                 ← Best ideas merged (updated after each round)
│   └── FINAL_SPEC.md                ← Locked architecture (built from SYNTHESIS.md)
│
└── /src                             ← Implementation begins here after FINAL_SPEC.md is locked
```

---

## 🚀 How This Works

1. Every AI reads `AI_INSTRUCTIONS.md` + `MASTER_BRIEF.md` before responding
2. Every AI saves its proposals to its own branch file in `/docs/ai-branches/`
3. After all AIs respond, Claude synthesizes into `SYNTHESIS.md`
4. All AIs challenge the synthesis (stress-test round)
5. Final architecture locked in `FINAL_SPEC.md`
6. Build begins

---

## 🎯 What We Are Building

An AI-native options trading system for **SPX/NDX/RUT index options** (Section 1256 / 60-40 tax treatment) with:

- **Dynamic regime detection** — the system knows what kind of market it's in
- **AI prediction engine** — direction, magnitude, timing forecasts
- **Intelligent strategy selection** — picks the optimal options strategy for each setup
- **Dynamic tiered capital allocation** — Core position + Satellites + Reserve
- **Iron-clad risk management** — protects capital as aggressively as it pursues profit
- **Continuous learning** — gets smarter with every single trade

**Broker:** Tradier API  
**Primary Mode:** 0DTE SPX options  
**Secondary Mode:** 1–5 day swing trades (regime-gated)  
**Tax Structure:** Section 1256 — 60% long-term / 40% short-term treatment

---

*Last updated: 2026 | Owner: tesfayekb*
