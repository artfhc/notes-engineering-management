# 1-Page Production Estimate — Hybrid “Fridge Landing Page” (Web-in-App)

[Replit App](https://coupang-fridge.replit.app)

## Goal
Ship a **production-ready category landing page** (like `/fridge`) consumed inside our **Android + iOS apps via WebView**, powered by **existing backend services**, optimized for **low latency**, and enabling **fast iteration** (no weekly release dependency).

---

## Constraints / Assumptions
- **Team:** 2 FE, 2 BE Engineers  
- **Hybrid allowed:** Web tech for the surface + native WebView shell (not React Native)
- **Iteration:** Web deploys + remote config/A-B tests enable daily iteration
- **Existing:** WebView surfaces exist; JS bridge exists (needs investigation); remote config/A-B framework exists
- **Infra:** CDN available for static assets and caching  

---

## Proposed Architecture (Latency-first)

### 1) FE Service (BFF-lite, FE-owned)
- Composes data from existing services into a **module-based page schema**
- **Timeouts + partial responses** (page renders even if some modules fail)
- Cache hints / TTL per module; **CDN stale-while-revalidate**

### 2) Web UI
- Mobile-first responsive page rendering modules:
  - Hero carousel
  - Product carousels (Deals / Best Sellers / Picks / Recommended)
  - Chips (People also search, Features)
  - Brand / Type tiles
  - Guides & Resources
  - Customer Q&A
  - Cross-sell
- **Figma MCP + Claude / Windsurf** for rapid CSS/UI implementation

### 3) Native Shell
- Reusable WebView container
- Minimal JS bridge:
  - auth/session
  - native navigation (PDP / search / login)
  - analytics & performance events
- Remote-config routing + kill switch

---

## MVP Scope vs Explicitly Out-of-Scope

### ✅ Included in MVP
- Fridge landing page rendered in WebView on **iOS + Android**
- Real data via existing backend services
- Core modules:
  - Hero carousel
  - 2–4 product carousels (Deals, Best Sellers, Picks, Recommended)
  - Chips (People also search, Features)
  - Shop by Type / Brand
  - Guides & Resources
  - Basic Customer Q&A (read + expand)
- Native deep-links:
  - Product → native PDP
  - Search / category → native search results
  - Login → native login
- Observability:
  - Page load timing
  - Module latency
  - Click & impression events
- Ops:
  - Remote-config routing
  - Kill switch
  - Rollback without app release

### ❌ Explicitly Out of Scope (Phase-2+)
- Checkout / payment / order flows
- Coupon redemption rules or eligibility enforcement
- Inventory reservation or delivery slot selection
- Fully native re-implementation of UI
- Advanced personalization logic (beyond existing recs service)
- Deep accessibility polish beyond baseline
- Pixel-perfect parity with Coupang production UI

---

## Delivery Timeline

### MVP (Production usable): **6–9 weeks**
**Definition of done**
- Page live in prod behind experiment
- Latency within agreed p95 budget
- Partial-failure safe (modules can drop without blank page)
- Analytics + rollback working

### V1 Polished: **9–12 weeks**
Adds:
- Tighter p95 latency
- Stronger caching & payload trimming
- Better impression attribution
- Accessibility & cross-device QA polish

---

## Week-by-Week Milestones (Indicative)

| Week | Milestone / Demo |
|-----:|------------------|
| 1 | WebView + JS bridge audit, page schema finalized, repos & CI set up |
| 2 | FE service skeleton + `/page/fridge` returns mocked modules |
| 3 | Web UI renders hero + 1 product carousel with real data |
| 4 | Native WebView integration; PDP/search deep-links working |
| 5 | Core modules complete; basic analytics events flowing |
| 6 | Latency tuning, caching enabled, partial-failure handling |
| 7–9 | QA, perf hardening, staged rollout via A/B + remote config |

---

## Preliminary Mobile Work (Foundational)
**~1–2 weeks (parallelizable)**

- Document existing WebView + JS bridge capabilities
- Define **Web Surface Contract** (params, events, versioning)
- Harden minimal bridge APIs:
  - `openProduct`
  - `openSearch`
  - `requestLogin`
  - `track`
  - `perf_report`
- Standardize WebView settings (domain allowlist, caching, crash recovery)
- Enable remote-config routing + kill switch

---
## Web Asset Shipping & Release Pipeline (Required for Production)

To safely ship and iterate on hybrid WebView pages, we need a **standard web asset pipeline** comparable to native app release discipline.

### Build & Bundling
- Use a modern build system (e.g. **Vite + Rollup** or **Webpack**) to produce:
  - Minified, tree-shaken JS/CSS
  - Code-split bundles (small initial payload, lazy-load below-the-fold)
- Enforce bundle size budgets for:
  - initial JS
  - initial CSS
  - above-the-fold JSON payload

### Asset Optimization
- Enable **Brotli / gzip compression** for JS, CSS, and JSON
- Optimize images:
  - serve responsive sizes (`srcset`)
  - prefer modern formats (WebP / AVIF where supported)
  - pre-compress hero and carousel images
- Extract critical CSS; avoid large runtime style libraries

### Versioning & Caching Strategy
- Use **hashed filenames** for immutable assets:
  - `app.<hash>.js`, `styles.<hash>.css`
- CDN configuration:
  - long TTL for hashed assets (days–months)
  - short TTL for HTML entry + page JSON (seconds–minutes)
  - **stale-while-revalidate** enabled
- Prevent stale-client / new-API mismatches via strict versioning

### Release, Rollout & Rollback
- Publish assets to CDN-backed storage (e.g. S3/GCS + CDN)
- Serve via **versioned entry URLs** or manifest
- Use **remote config / A-B framework** to:
  - control rollout by cohort
  - switch versions instantly
  - roll back to last known good without app release
- Keep at least one previous production version live for emergency rollback

### Runtime Compatibility & Safety
- Host **source maps** privately or behind auth for debugging
- Implement bridge version negotiation:
  - web detects supported bridge version
  - gracefully degrades if APIs are unavailable
- Feature flags in web align with native experiment keys

### Monitoring & Observability
- Track:
  - asset download size & time
  - WebView page load timings (p50 / p95)
  - JS errors tagged with release version
  - CDN cache hit ratio
- Alert on sudden regressions in payload size or load time

### Estimated Setup Cost
- If existing web infra templates exist: **2–5 days**
- If building pipeline from scratch: **~1 week**
- Included in overall MVP estimate

> **Definition of Done:**  
> A web release can be built, shipped, cached, rolled out, and rolled back independently of mobile app releases, with measurable performance and safety guarantees.


---

## Key Risks & Mitigations
- **Upstream latency / fan-out** → FE service composition + timeouts + CDN SWR
- **Hybrid scroll jank** → early perf instrumentation, image & payload optimization
- **Bridge drift** → versioned bridge + schema validation
- **Mobile → Web ramp-up** → plan 1.2–1.5× buffer; rely on AI for scaffolding, not architecture

---

## Staffing Ownership
- **Mobile Eng A:** Web UI + FE service + caching + schema/versioning  
- **Mobile Eng B:** Native WebView shell + JS bridge + auth/nav + perf hooks  
- **Backend Engs:** Upstream service enablement, data correctness, SLA & latency support

---

## Executive Summary
Using a **hybrid WebView approach**, FE-owned composition service, and **AI-assisted development**, we can deliver a **production-ready Fridge landing page MVP in ~6–9 weeks** (polished V1 in **~9–12 weeks**) while preserving latency guarantees and enabling rapid iteration without app-store release delays.


## Risks & Mitigations

### 1) Hybrid WebView Performance Risk
**Risk:**  
Image-heavy carousels, long lists, and nested scrolling may cause scroll jank, memory pressure, or slow first render in WebView (especially on lower-end devices).

**Mitigation:**  
- Enforce a **performance budget** (payload size, image sizes, number of above-the-fold modules)
- Use **single composed page payload** with per-module timeouts
- Lazy-load below-the-fold modules and images
- Instrument WebView load milestones early (p50 / p95 tracking)
- Roll out behind A/B with kill switch

---

### 2) FE Service (BFF-lite) Complexity
**Risk:**  
Composing multiple existing backend services can introduce fan-out latency, brittle contracts, or operational overhead for a FE-owned service.

**Mitigation:**  
- Keep the FE service **read-only and stateless**
- Enforce **strict timeouts + partial responses** per module
- Cache aggressively via CDN (short TTL + stale-while-revalidate)
- Version the page schema to allow safe iteration
- Backend engineers support upstream contract simplification and SLAs

---

### 3) JS Bridge Drift & Platform Inconsistency
**Risk:**  
Inconsistent or undocumented JS bridge APIs can cause subtle behavior differences between iOS and Android and slow down web iteration.

**Mitigation:**  
- Define a **minimal, versioned bridge contract** (v1)
- Schema-validate all bridge messages
- Ensure web pages degrade gracefully if bridge APIs are unavailable
- Centralize bridge ownership and documentation

---

### 4) Mobile Engineers Ramping into Web
**Risk:**  
Mobile engineers new to web tooling may underestimate CSS/layout edge cases, WebView quirks, or FE service operational concerns.

**Mitigation:**  
- Use **Claude Code / Windsurf + Figma MCP** for scaffolding and layout iteration
- Start with a **limited set of reusable modules**
- Plan a **1.2–1.5× buffer** in early estimates
- Lean on backend engineers for infra and observability best practices

---

### 5) Latency Regressions from Upstream Services
**Risk:**  
Even with a fast FE service, slow upstream APIs (deals, recs, Q&A) can degrade overall page latency.

**Mitigation:**  
- Define **module-level SLAs**
- Fail open: render the page without slow modules
- Cache hot modules independently
- Monitor per-module latency, not just page-level metrics

---

### 6) Over-expansion of MVP Scope
**Risk:**  
Pressure to add checkout, coupon logic, or deep personalization can turn the landing page into a full commerce flow and delay launch.

**Mitigation:**  
- Lock MVP scope explicitly (landing + discovery only)
- Defer transactional logic to native flows
- Use feature flags to gate experimental modules
- Revisit scope only after MVP latency and stability targets are met

---

### 7) Long-Term Migration Cost
**Risk:**  
If the page later needs to be fully native, hybrid implementation may introduce migration cost.

**Mitigation:**  
- Treat this surface as a **content/discovery page**
- Keep business logic in backend services, not web
- Define clear ownership boundaries so native reimplementation is possible if needed