# Tech Lead Interview Guide — Fridge Prototype

**Goal:**  
Understand what in the prototype is *throwaway vs reusable*, surface hidden assumptions, and identify production gaps that affect estimation.

---

## 1) Architecture & Stack
**Intent:** Identify what was built for speed vs what can scale.

**Questions**
- What stack did you use for the web UI (framework, bundler, CSS)?
- Is this a static page or a SPA with client-side routing?
- How are pages like `/fridge`, `/product/view`, and guides wired together?
- Where is state stored (URL, JS memory, localStorage, none)?
- Are there any hardcoded assumptions (category IDs, currency, locale)?

**Listen for**
- “Hardcoded”, “mocked”, “client-side only”, “not designed to scale”

---

## 2) Data Sources & Contracts
**Intent:** Understand backend integration gaps.

**Questions**
- Which data is real vs mocked?
- How many distinct data sources are involved?
- Are requests parallel or sequential?
- Do you handle timeouts or partial failures?
- What happens if one source is slow or fails?

**Listen for**
- Browser-side fan-out
- No timeout handling
- Assumptions that all services succeed

---

## 3) Performance & Latency
**Intent:** Surface scale-related blind spots.

**Questions**
- Have you measured page load time or bundle size?
- What renders above the fold first?
- Are images optimized or just embedded?
- Any lazy loading or code splitting?
- Tested on lower-end devices?

**Listen for**
- “Didn’t measure”
- “Fast on my machine”
- No CDN or compression

---

## 4) WebView & Native Integration
**Intent:** Detect browser vs in-app mismatches.

**Questions**
- Was this designed for browser or WebView first?
- How should navigation to PDP/search work in-app?
- How do you expect auth/session to work?
- Any browser APIs that may not work in WebView?
- Have you tested inside an actual app WebView?

**Listen for**
- Browser-only assumptions
- No auth or JS bridge awareness

---

## 5) Deployment & Iteration Model
**Intent:** Identify missing production infrastructure.

**Questions**
- How is this deployed today?
- Is there a build step or live editing?
- How would you roll back a bad change?
- Are assets versioned and cache-busted?
- How would you run A/B tests?

**Listen for**
- “Replit auto-deploys”
- No rollback/versioning story

---

## 6) Ownership & Maintenance
**Intent:** Align on long-term expectations.

**Questions**
- Who do you expect to own this long-term?
- Is this prototype-only or a foundation?
- What parts would you not ship as-is?
- What would you redo with more time?
- What do you think will be hardest to productionize?

**Listen for**
- “Cut corners here”
- “Never meant to scale”

---

## 7) Explicit Cut-Corners Question (Most Important)
**Ask directly**
> If this had to go to production with real traffic, what are the top 3 things you’d redo or worry about first?

---

## 8) Estimation Alignment
**Close with**
> Does a 6–9 week estimate for a production-grade version feel reasonable?  
> What would push it shorter or longer?

---

## What You’re Really Validating

| Area | What It Tells You |
|------|-------------------|
| Stack | Rebuild vs reuse |
| Data | BFF necessity |
| Performance | Hidden latency work |
| WebView | Mobile feasibility |
| Deployment | Missing infra |
| Scope | Prototype shortcuts |
