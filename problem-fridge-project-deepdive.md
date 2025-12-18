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