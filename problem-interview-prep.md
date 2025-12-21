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
- Timeouts, circuit breakers, bulkheads  
- Rate limiting, load shedding  
- Consistency models (strong vs eventual)  
- Data modeling with growth in mind (indexes, hot partitions)

**Practice designs:**
- Donation service (DoorDash-style)
- Messaging system (Lyft-style TPS)

**Deliverable:**
- Reusable **Reliability Appendix** you can add to any design in ~2 minutes

---

### Week 3 — Boundaries & Ownership: “Who Builds What”

**Goal:** Fix the “hard to define system boundaries/team ownership” gap.

**Always include:**
- Domain boundaries (auth, catalog, payments, messaging, search, etc.)
- Platform boundaries (SDKs, design system, CI/CD, observability)
- Team interfaces (APIs, SLAs, oncall ownership)

**Practice scenarios:**
- LLM chat UI with carousel / rich cards
- Mobile monolith modernization

**Deliverable:**
- **Team Topology Template**
  - Teams
  - Ownership
  - Contracts
  - Rollout strategy

---

### Week 4 — Sharding + Scaling Decisions (Weakest Area)

**Goal:** Confidently answer “how does this scale from 10× to 100×?”

**Topics:**
- Sharding strategies (user_id, tenant_id, region, time)
- Hot key mitigation (salting, consistent hashing, caching)
- Read/write separation, CQRS (when useful vs overkill)
- Partitioned queues/streams + consumer scaling

**Practice designs:**
- File storage (Dropbox-style)
- News feed (Meta/Google-style)

**Deliverable:**
- **Scaling Playbook**
  - Bottleneck  
  - Symptoms  
  - Fixes  
  - Trade-offs  

---

### Week 5 — Mobile Architecture (Director-Level)

**Goal:** Nail mobile infra, velocity, and quality angles.

**Topics:**
- Modularization strategy (feature vs core modules, dependency rules)
- Build time reduction (CI, caching, Gradle config, parallelism)
- Testing pyramid (unit, integration, screenshot, E2E)
- Release safety (canary, feature flags, kill switches, staged rollout)

**Practice:**
- 10-year mobile monolith → phased migration plan

**Deliverable:**
- **30/60/90-Day Plan** for joining a messy mobile org

---

### Week 6 — Coding + Execution Consistency

**Goal:** Reduce unforced errors and cognitive drift.

**Practice (6–8 sessions):**
- Trees (BST / BT transforms like tree → DLL)
- Maps + sorting + streaming data
- Concurrency basics (producer/consumer, rate limiter)

**Rules:**
- Always write invariants at the top
- Always run through 2 examples manually
- Always state time/space complexity

**Extra:**
- 1 intentional **debug session** per week (break code → fix it)

**Deliverable:**
- Personal **Coding Completion Checklist**

---

### Week 7 — Company Targeting & Packaging

**Goal:** Tailor without reinventing yourself.

**Build 4 versions of:**
- Why this company
- Why this org/team
- Your unique edge

**Company focus areas:**
- **Anthropic:** product judgment, safety, execution under ambiguity
- **Discord:** real-time systems, community safety, platform velocity
- **Meta / Google:** scale, experimentation rigor, crisp system design

**Deliverable:**
- 1-page **Talk Track Sheet** per company

---

### Week 8 — Full Mock Loop

**Goal:** Simulate onsite performance.

**Per week:**
- 2 full mock loops:
  - 1 system design (60 min)
  - 1 behavioral (45 min)
  - 1 manager + execution (45 min)

**After each mock:**
- Write the **5 bullets** you wish you had said

**Deliverable:**
- **Top 20 Fixes List**
- Redo those exact questions

---

## Weekly Routine (Repeatable)

### Mon–Fri (60–90 min/day)
- 10 min — Story rehearsal (1 STAR)
- 35–45 min — System design drill (timed)
- 15–30 min — Coding or scaling/reliability flashcards

### Saturday (2–3 hours)
- 1 full system design mock (record yourself)
- 30 min review:
  - Missing failure modes
  - Data model gaps
  - Ownership clarity

### Sunday (60–90 min)
- Behavioral pack:
  - 3 questions
  - 7 minutes each
- Update story bank + company talk tracks

---

## Default Answer Frameworks

### 1) System Design Talk Track (Use Every Time)

1. Requirements (functional + non-functional + constraints)  
2. Core entities + APIs  
3. High-level architecture (clients → edge → services → data/queues)  
4. Data model + access patterns  
5. Scaling plan (what breaks first?)  
6. Failure modes + reliability (idempotency, retries, DLQs, monitoring)  
7. Rollout plan + team ownership  

> If you do only this, your answers will stop feeling messy.

---

### 2) Team / Ownership Framework

**Split by rate of change and blast radius:**
- Product feature teams (fast iteration)
- Platform teams (shared infra, stable contracts)
- Enabling teams (CI/CD, test infra, observability)

**Define explicitly:**
- Interfaces (APIs / SDK contracts)
- SLOs (latency, crash-free rate, build time)
- Oncall ownership (who gets paged)

---

## Top Priorities (Based on Your Journal)

1. **Behavioral clarity** — STAR structure + crisp results  
2. **Reliability & failure modes** — add proactively, not after prompting  
3. **Ownership & boundaries** — recurring weakness across rounds  
4. **Coding consistency** — stability > perfection
