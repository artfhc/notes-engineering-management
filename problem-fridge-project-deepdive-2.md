# 1-Page Production Estimate — Hybrid “Fridge Landing Page” (Web-in-App)

[Prototype (Replit)](https://coupang-fridge.replit.app)

---

## Goal
Deliver a **production-ready category landing page** (e.g., `/fridge`) rendered inside **Android + iOS apps via WebView**, backed by **existing backend services**, optimized for **low latency**, and enabling **daily iteration** without app releases.

---

## Scope Summary (What we’re building)
A **discovery-first landing page** composed server-side and rendered via web modules, with native deep-links for transactional flows (PDP/search/login). This is **not** checkout or coupon logic.

---

## Team, Constraints & Assumptions
- **Team:** 2 FE, 2 BE engineers  
- **Hybrid:** Web UI in WebView + native shell (not React Native)  
- **Iteration:** Web deploys + remote config/A-B for rollout/rollback  
- **Existing:** WebView surfaces, JS bridge (needs audit), A/B infra  
- **Infra:** CDN available for static assets + caching  

**Key assumptions (estimate depends on these):**
- Upstream APIs are readable and stable (no new business logic)
- Auth/session plumbing already exists
- Page remains **discovery-only**
- AI tooling (Claude/Windsurf + Figma MCP) used for scaffolding

> If any assumption breaks, expect **+1–3 weeks** depending on severity.

---

## Architecture (Latency-First)

### 1) FE Service (BFF-lite, FE-owned)
- Single page endpoint (e.g., `GET /page/fridge`)
- **Module-based page schema** (renderer-friendly, reorderable, experimentable)
- **Per-module timeouts + partial responses** (page renders even if modules fail)
- Cache hints per module; CDN **stale-while-revalidate**
- **Read-only and stateless** (no commerce logic)

### 2) Web UI
- Mobile-first module renderer:
  - Hero carousel
  - Product carousels (Deals / Best Sellers / Picks / Recommended)
  - Chips (People also search / Features)
  - Brand & Type tiles
  - Guides & Resources
  - Customer Q&A
  - Cross-sell
- Built with modern tooling (Vite/Webpack), AI-assisted CSS/UI

### 3) Native Shell
- Reusable WebView container
- **Minimal JS bridge (v1)**:
  - auth/session
  - native navigation (PDP / search / login)
  - analytics & performance events
- Remote-config routing + kill switch

---

## MVP Scope vs Out of Scope

### ✅ MVP (Production-Usable)
- WebView page live on **iOS + Android**
- Real data from existing backend services
- Core modules listed above
- Native deep-links to PDP/search/login
- **Observability:** page load timing, module latency, clicks/impressions
- **Operations:** experiment rollout, instant rollback (no app release)

### ❌ Out of Scope (Phase-2+)
- Checkout, payment, coupon enforcement
- Inventory reservation or delivery slots
- Fully native re-implementation
- Advanced personalization beyond existing recs
- Pixel-perfect parity; deep accessibility polish

---

## Delivery Timeline (Confidence-Based)

### MVP: **6–9 weeks**
- **6 weeks (best case):** stable APIs, clean bridge, minimal churn  
- **9 weeks (realistic):** perf tuning, bridge cleanup, QA iterations

**Definition of Done**
- Live behind experiment
- Meets p95 latency budget
- Partial-failure safe
- Rollback verified without app release

### V1 Polished: **9–12 weeks**
- Tighter p95 latency
- Payload trimming & cache tuning
- Improved impression attribution
- Accessibility & cross-device polish

---

## Effort Breakdown & Critical Path

### Effort Breakdown (MVP)

| Workstream | Owner | Scope | Est. Effort |
|----------|------|-------|-------------|
| FE Service (BFF-lite) | FE | Schema, fan-out, timeouts, caching, CDN | 2–3 weeks |
| Web UI | FE | Modules, layout/CSS, perf, analytics | 2–3 weeks |
| Native WebView Shell | Mobile | Container, routing, auth, error handling | 1–2 weeks |
| JS Bridge Hardening | Mobile | Versioned APIs, schema validation | 1–2 weeks |
| Web Asset Pipeline | FE | Build, bundling, compression, rollback | ~1 week |
| Observability & Perf | FE + Mobile | p50/p95 metrics, error logging | ~1 week |
| QA & Rollout | All | Device QA, A/B rollout, rollback drills | 1–2 weeks |

> Most work runs in parallel; the **critical path** determines total duration.

### Critical Path (Timeline Drivers)
1. Page schema finalization (blocks FE service & Web UI)
2. JS bridge v1 stabilization (blocks native deep-links)
3. Web asset shipping & rollback pipeline (blocks prod rollout)
4. Latency tuning & caching (blocks prod readiness)

---

## Web Asset Shipping & Release (Required)

- **Build & Bundling:** Vite/Webpack; minified, tree-shaken, code-split
- **Optimization:** Brotli/gzip; responsive images (WebP/AVIF); critical CSS
- **Versioning & Caching:** hashed assets (long TTL); HTML/JSON short TTL + SWR
- **Release/Rollback:** versioned entry URLs; A/B-driven rollout; instant rollback
- **Monitoring:** asset size/time, p50/p95, JS errors (version-tagged), CDN hit rate

**Setup Cost:**  
- Existing templates: **2–5 days**  
- From scratch: **~1 week** (included in estimate)

---

## Key Risks & Mitigations

- **Hybrid performance/jank:** enforce perf budgets, lazy loading, early instrumentation
- **FE service fan-out latency:** strict timeouts, partial rendering, CDN SWR
- **JS bridge drift:** versioned contract + schema validation
- **Mobile → Web ramp-up:** limited module set, AI-assisted scaffolding
- **Scope creep:** lock discovery-only MVP

---

## Ownership
- **FE A:** Web UI + FE service + caching/schema/versioning  
- **Mobile B:** WebView shell + JS bridge + auth/nav + perf hooks  
- **BE:** Upstream APIs, data correctness, SLAs & latency support

---

## Executive Summary
With a **hybrid WebView approach**, an FE-owned **BFF-lite composition layer**, and **AI-assisted delivery**, we can ship a **production-ready Fridge landing page MVP in ~6–9 weeks** (polished V1 in **~9–12 weeks**) while meeting latency goals and enabling rapid iteration without app-store delays.
