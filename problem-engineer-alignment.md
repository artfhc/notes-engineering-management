
# Management Action Plan – Addressing Team Feedback from Mike

This document captures **both the problems raised and the concrete solutions / action plans from a management perspective**.  
All actions are **EM-owned**, practical, and designed to improve clarity, quality, and execution predictability without adding heavy process.

---

## Management Philosophy Behind the Plan

- Ambiguity is more expensive than slow decisions
- Starting fast is worse than starting clear
- Quality failures are management problems before they are engineering problems
- Repeated issues indicate missing system-level guardrails

---

## Problem → Solution / Action Plan Overview

| # | Problem Area | Core Issue | Management Solution / Action Plan | Outcome |
|---|-------------|-----------|-----------------------------------|---------|
| 1 | Unclear project goals & context | Engineers don’t understand *why* they are building something | Introduce mandatory project framing, re-brief on priority changes, and assign metric ownership | Better decision-making, less thrash |
| 2 | Requirements not finalized early | Work starts with unresolved legal / UX / product constraints | Add a Ready-to-Build gate and track known unknowns explicitly | Fewer mid-project pivots |
| 3 | UX bottleneck | UX decisions land too late | Front-load directional UX decisions and assign a single decision owner | Backend & FE unblock earlier |
| 4 | Repeating issues across projects | No feedback loop to fix systemic problems | Run lightweight process post-mortems and track recurring “project smells” | Issues fixed once, not repeatedly |
| 5 | iOS quality & QA gaps | QA is compressed or skipped under deadline pressure | Enforce QA windows and trade scope for quality | Fewer prod bugs |
| 6 | Backend dependency issues | FE blocked by unstable backend environments | Require backend readiness before FE QA and assign env ownership | Predictable integration |
| 7 | Weak code review bar | Reviews prioritize speed over correctness | Reset review expectations and rotate strict reviewers | Higher code quality |
| 8 | Constant rush & unclear growth path | Pace prioritized over clarity and development | Clarify roles, expectations, and growth signals per project | Higher trust & retention |

---

## Detailed Solutions & Action Items

### 1. Unclear Project Goals & Context
**Actions**
- Require a short **Project Framing** before work starts:
  - Goal (1 sentence)
  - Primary metric
  - Why now (priority rationale)
  - Explicit non-goals
- Any priority change requires a **re-brief**
- Assign a clear **metric owner**

**Result**
Engineers understand intent and make better tradeoffs independently.

---

### 2. Requirements Not Finalized Early
**Actions**
- Introduce a **Ready-to-Build** checklist (UX direction, legal dependencies identified)
- Maintain a **Known Unknowns** list with owners and deadlines
- Push back on demo-driven scope changes unless leadership re-prioritizes

**Result**
Work starts with eyes open instead of assumptions.

---

### 3. UX as a Bottleneck
**Actions**
- Require **directional UX approval** before sprint start
- Assign a single **UX decision owner**
- Time-box UX discussions with a default fallback

**Result**
Execution is no longer blocked by open-ended UX debates.

---

### 4. Same Problems Repeating
**Actions**
- Run a 15-minute **process post-mortem** after each project
- Maintain a running list of **project smells**
- Escalate early when the same smells appear

**Result**
Systemic issues are addressed once instead of rediscovered.

---

### 5. iOS Quality & QA Gaps
**Actions**
- Enforce a **non-negotiable QA window**
- Make scope vs quality tradeoffs explicit and visible
- Assign a **release quality owner**

**Result**
Quality improves without heroics.

---

### 6. Backend Environment & Dependency Issues
**Actions**
- Require backend confirmation before FE QA starts
- Add backend readiness to **go / no-go** decisions
- Make environment ownership explicit

**Result**
Fewer last-minute integration failures.

---

### 7. Weak Code Review Bar
**Actions**
- Publicly reset expectations for code reviews
- Rotate a **strict reviewer** role each sprint
- Call out strong review behavior in retros

**Result**
Issues caught earlier and standards raised.

---

### 8. Constant Rush & Unclear Growth Path
**Actions**
- Start each project with **role clarity**
- Tie projects explicitly to **career growth signals**
- Deliberately choose some initiatives to prioritize correctness over speed

**Result**
Higher morale, better long-term performance.

---

## Leadership Framing (Optional)

> “These changes are about removing ambiguity early so we can move faster *and* with higher quality, without burning out the team.”

---

## How to Use This Document
- Share selectively with the team to explain **new expectations**
- Use to show **proactive risk reduction**
- Treat as a checklist when kicking off projects

