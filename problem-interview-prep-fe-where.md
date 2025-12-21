# Exactly Where to Learn Frontend + Mobile Architecture (EM Track)

This list is **opinionated and concrete**:
- Specific sites
- Specific sections to read
- What to skim vs internalize
- Why each matters for interviews

You do NOT need to finish everything.
You DO need to recognize the concepts and talk about trade-offs fluently.

---

## 1️⃣ Frontend Rendering, SSR/CSR, Hydration (Foundational)

### ✅ START HERE (Highest ROI)
**:contentReference[oaicite:0]{index=0}**

👉 Go to:
- web.dev/learn/
- web.dev/vitals/

**Read (don’t skip):**
- Core Web Vitals (LCP, CLS, INP)
- Critical Rendering Path
- Rendering on the Web (SSR vs CSR vs hybrid)

⏱ Time: ~2–3 hours total

**Interview payoff:**
You’ll sound immediately credible when discussing performance and first paint.

---

### Secondary (Interview Language Borrowing)
**:contentReference[oaicite:1]{index=1} Blog**

👉 Go to:
- vercel.com/blog

Search:
- “SSR vs CSR”
- “Streaming”
- “Partial rendering”

Skim — don’t deep dive.

**Why:**  
Meta / Google / Discord interviewers use this language even if they don’t use Next.js.

---

## 2️⃣ Frontend Performance & Debugging (Very High Signal)

### ✅ Must-Watch
**:contentReference[oaicite:2]{index=2} (YouTube)**

👉 Search on YouTube:
- “Chrome DevTools performance profiling”
- “Understanding the performance tab”

Watch **1–2 videos**, not all.

**Interview payoff:**
You can explain *where* slowness comes from and *how* you’d debug it.

---

## 3️⃣ State Management & Data Fetching (EM-Level Only)

### ✅ Read Concepts (NOT APIs)

**:contentReference[oaicite:3]{index=3}**
- tanstack.com/query/latest/docs/framework/react/overview

**:contentReference[oaicite:4]{index=4}**
- swr.vercel.app

Focus ONLY on:
- Server state vs UI state
- Cache invalidation
- Optimistic updates

Ignore:
- Hooks syntax
- Advanced configs

⏱ Time: ~60–90 min

**Interview payoff:**
You can say:
> “Most frontend complexity comes from mismanaging server state.”

---

## 4️⃣ Design Systems & Component Ownership (Common EM Question)

### ✅ Read This Book (Skim Strategically)
**:contentReference[oaicite:5]{index=5}**

Read chapters on:
- Ownership
- Versioning
- Adoption problems

Skip:
- Visual design details

⏱ Time: ~2 hours skim

---

### Real-World Systems (Pick ONE)
- **:contentReference[oaicite:6]{index=6}**
- **:contentReference[oaicite:7]{index=7}**

Focus on:
- How components evolve
- Backward compatibility
- Team autonomy

**Interview payoff:**
You can discuss when design systems *slow teams down*.

---

## 5️⃣ Frontend Architecture & Boundaries (EM Thinking)

### ✅ Boundary Thinking (Not Implementation)
**:contentReference[oaicite:8]{index=8}**

Read:
- Intro
- Boundary & ownership chapters

Skip:
- Webpack / implementation details

⏱ Time: ~1.5 hours skim

**Why this matters:**
Even if you never use micro-frontends, interviewers care about **blast radius and ownership**.

---

## 6️⃣ Frontend Reliability, Rollout & Kill Switches

### ✅ High-Signal Engineering Blogs

**:contentReference[oaicite:9]{index=9}**
- Search:
  - “frontend resilience”
  - “canary releases”
  - “kill switch”

**:contentReference[oaicite:10]{index=10}**
- Search:
  - “frontend architecture”
  - “progressive delivery”

Skim articles — don’t binge.

**Interview payoff:**
You’ll talk about frontend incidents like backend incidents.

---

## 7️⃣ Hands-On Without Coding (Best EM Exercise)

### Weekly Practice (15–30 min/session)

Pick any site:
- Amazon
- Airbnb
- Discord Web

Open DevTools and ask:
1. What renders first?
2. What blocks LCP?
3. Where is data fetched?
4. What is cached?
5. What breaks on slow network?
6. How would I roll this out safely?

Do this **twice a week**.

**This builds intuition faster than tutorials.**

---

## 8️⃣ If You Only Have 1 Week (Strict Priority)

1. web.dev (Core Web Vitals + rendering)
2. Chrome DevTools performance videos
3. React Query concepts
4. Design Systems (ownership chapters)
5. One Netflix frontend article

That’s it.

---

## EM Mental Model (Memorize This)

> Frontend interviews are not about React.
> They are about **performance, boundaries, reliability, and team velocity**.

---

## What We Can Do Next (Highly Recommended)

- Build a **1-page frontend interview cheat sheet**
- Run a **mock frontend system design** (Meta / Google style)
- Convert **one of your mobile designs** into a frontend-first answer
- Create a **Week-5 daily study checklist with links**

Tell me which one you want — and I’ll make it concrete.
