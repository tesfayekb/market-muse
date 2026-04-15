# AI_INSTRUCTIONS.md — Universal Rules for All AI Contributors
**Applies to:** Claude, GPT-4o, Gemini, Grok, and any future AI contributor  
**Repo:** https://github.com/tesfayekb/market-muse.git  
**Read this before reading anything else.**

---

## 🔴 THE PRIME DIRECTIVE

**PROFIT IS THE ONLY GOAL.**

Not elegance. Not complexity. Not academic interest. Not technical novelty.  
**Profit.**

Every single proposal you make must answer this question first:

> *"How does this directly increase the net after-tax profitability of this trading system?"*

If you cannot answer that question clearly and specifically for an idea — do not include it.  
If an idea is theoretically interesting but does not increase profit — discard it.  
If a simpler approach is more profitable than a complex one — choose simple.  
**Profit is the filter. Apply it to everything.**

---

## 📋 YOUR MANDATORY READING LIST

Before writing a single word of your proposal, you must read these files in order:

1. `README.md` — understand the project structure
2. `MASTER_BRIEF.md` — understand the full system specification
3. `/docs/ai-branches/` — read ALL existing AI proposals (do not duplicate, build on them)
4. `/docs/SYNTHESIS.md` — if it exists, read the current best thinking

**Do not skip any of these. Do not skim. Read completely.**

---

## 📐 MANDATORY RESPONSE STRUCTURE

Your response must be saved to `/docs/ai-branches/[your-ai-name]-branch.md`  
Follow this structure exactly — do not invent your own format:

```markdown
# [AI Name] — Branch Proposal v[X]
**Date:** [Date]
**Reading from:** MASTER_BRIEF v[version]
**Prior branches reviewed:** [list which ai-branch files you read]

---

## 💡 MY SINGLE HIGHEST-LEVERAGE PROFIT IDEA
[One paragraph. The single idea you believe will have the greatest impact 
on profitability. Be specific. Quantify the expected impact if possible.]

---

## PILLAR 1: Prediction Engine
### My Proposed Architecture
### Why This Maximizes Profit
### Specific Implementation Steps

## PILLAR 2: Strategy Selection Engine  
### My Proposed Architecture
### Why This Maximizes Profit
### Specific Implementation Steps

## PILLAR 3: Risk Management Engine
### My Proposed Architecture
### Why This Maximizes Profit
### Specific Implementation Steps

## PILLAR 4: Monitoring & Dashboard
### My Proposed Architecture
### Why This Maximizes Profit
### Specific Implementation Steps

## PILLAR 5: P&L, Stop-Loss & Exit Strategy
### My Proposed Architecture
### Why This Maximizes Profit
### Specific Implementation Steps

## PILLAR 6: AI Learning & Adaptation
### My Proposed Architecture
### Why This Maximizes Profit
### Specific Implementation Steps

---

## 🛠️ TECHNICAL STACK RECOMMENDATIONS
[Specific technologies for each component with justification]

---

## ⚠️ CRITICAL RISKS & BLIND SPOTS
[What in the current plan could cause failure or destroy profit]

---

## 🔁 RESPONSES TO OPEN QUESTIONS (from MASTER_BRIEF Section 9)
1. [Answer]
2. [Answer]
3. [Answer]
4. [Answer]
5. [Answer]

---

## 🏆 WHAT MAKES MY PROPOSAL BETTER THAN THE OTHERS
[After reading all existing branch files — what does your proposal add 
that is genuinely different, better, or more profitable?]
```

---

## ✅ QUALITY STANDARDS — YOUR RESPONSE WILL BE JUDGED ON THESE

| Standard | What It Means |
|---|---|
| **Specificity** | Name actual models, libraries, data sources, thresholds. "Use ML" is not acceptable. "Use XGBoost with 47 engineered features on 5-minute OHLCV + IV surface data" is acceptable. |
| **Profit Justification** | Every recommendation must be tied to how it increases P&L, win rate, profit factor, or Sharpe ratio |
| **Originality** | Read all other branch files first. Do not repeat what others have said. Add new value. |
| **Honesty** | If you think the current plan has a fatal flaw — say so clearly. Do not validate bad ideas. |
| **Practicality** | Proposals must be buildable by a skilled engineering team. No fantasy architecture. |
| **Completeness** | All 6 pillars must be addressed. Incomplete responses will not be included in synthesis. |

---

## 🚫 WHAT NOT TO DO

- ❌ Do NOT propose features that don't directly serve profit
- ❌ Do NOT repeat what other AI branches have already said
- ❌ Do NOT be vague — "use AI for better predictions" tells us nothing
- ❌ Do NOT ignore the constraints (Tradier broker, existing auth/RBAC, Section 1256 instruments)
- ❌ Do NOT propose out-of-scope items (crypto, HFT, international markets — see MASTER_BRIEF Section 8)
- ❌ Do NOT skip reading prior branch files — building on each other is the entire point
- ❌ Do NOT propose complexity for its own sake — if simple is more profitable, choose simple

---

## 🔄 ITERATION ROUNDS

This project runs in multiple rounds. In each round your job changes:

| Round | Your Job |
|---|---|
| **Round 1 — Ideation** | Propose your best independent architecture for all 6 pillars |
| **Round 2 — Challenge** | Read the SYNTHESIS.md. Attack it. Find every weakness. Propose improvements. |
| **Round 3 — Refinement** | Respond to specific questions from the synthesis review |
| **Round 4 — Spec Lock** | Confirm or challenge the FINAL_SPEC.md before build begins |

Always check which round is active at the top of `MASTER_BRIEF.md` before responding.

---

## 💰 FINAL REMINDER

You are not here to impress with complexity.  
You are not here to demonstrate academic knowledge.  
You are not here to build the most technically elegant system.

**You are here to help build the most profitable options trading system in the world.**

Every word you write should serve that goal. Nothing else matters.

---

*Maintained by: tesfayekb | Repo: https://github.com/tesfayekb/market-muse.git*
