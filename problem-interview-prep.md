# EM Interview Ramp Plan (8 Weeks)

This plan is built around the gaps that show up repeatedly in your interview journal:

- **System design depth:** sharding, failure modes, edge cases, scaling trade-offs  
- **System boundaries / team ownership:** defining “who owns what” and why  
- **Decision clarity:** crisp “options → decision → justification with constraints”  
- **Behavioral/storytelling:** STAR clarity + stronger “why me / why this company / why management”  
- **Coding reliability:** fewer unforced errors (global state, invariants, testing, step-by-step)

This plan includes:
1. A weekly roadmap  
2. A repeatable weekly routine  
3. Answer frameworks for your most common misses  

---

## 8-Week Roadmap (EM-Focused, Mobile-Friendly, Company-Agnostic)

### Week 1 — Baseline + Story Bank + Core Narratives

**Goal:** Eliminate “messy answers” by having ready primitives.

**Write 8 STAR stories and rehearse them:**
- Conflict with XFN
- Low performer / coaching
- Hiring + raising the bar
- Outage / incident + prevention
- Big technical bet (SDUI / LLM UX / Visual Search)
- Execution under ambiguity
- Disagree-and-commit
- Failure + what changed afterward

**Create 3 topline narratives (90 seconds each):**
- Personal background *(called out as weak)*
- Why EM *(called out as weak)*
- Operating philosophy (quality, customer impact, velocity, people)

**Deliverable:**
- One-page **Story Index**  
  - Title  
  - 3 bullets  
  - Metric / result  
  - What you learned  

---

### Week 2 — System Design Foundations: Reliability + Scale Muscle Memory

**Goal:** Talk about failure and edge cases *without being prompted*.

**Drill until default:**
- Idempotency keys, retries/backoff, DLQs  
- Timeouts, ci
