# Week 5 Expansion — Frontend Skillset for EM Interviews

This expands **Week 5** of your EM ramp plan to explicitly cover **frontend architecture**, while staying aligned with mobile, platform, and execution expectations at **Meta / Google / Discord / Anthropic** level.

---

## What “Good” Looks Like in Frontend EM Interviews

You are **not** expected to be a React expert. You **are** expected to:

- Make clear frontend architecture decisions
- Understand performance trade-offs
- Design scalable UI systems (design systems, state, data flow)
- Reason about experimentation, reliability, and rollout
- Connect frontend choices to team velocity and product impact

**Mental model:**  
> “I don’t write all the code, but I can design the system, review it critically, and unblock teams.”

---

## 1. Frontend Fundamentals (Non-Negotiable)

You should be able to explain these **clearly and confidently**.

### Core Concepts
- SPA vs MPA trade-offs
- Client-side rendering (CSR) vs server-side rendering (SSR)
- Hydration: what it is, why it fails, performance implications
- Virtual DOM vs compilation-based rendering
- State ownership: local vs global vs server state

### Performance
- Critical rendering path
- JavaScript bundle size vs code splitting
- Network waterfalls
- Caching (HTTP cache, CDN, in-memory)
- Lazy loading (routes, images, components)

### Data & State
- Fetch-on-render vs fetch-then-render
- Server state vs UI state (React Query / SWR mental model)
- Optimistic updates
- Error and loading states as first-class UX

**Interview signal:**  
If you can explain *why* something is slow and *where* you’d instrument, you’re ahead of most EMs.

---

## 2. Frontend Architecture Topics (EM Level)

These are the areas interviewers probe **implicitly**.

### a) Frontend Modularization
- Feature-based vs layer-based structure
- Shared UI libraries vs app-specific components
- Dependency rules to avoid frontend spaghetti
- Versioning and rollout of shared components

### b) Design Systems at Scale
- Design tokens (color, spacing, typography)
- Component ownership model
- Backward compatibility
- How design systems accelerate teams *and* create friction

### c) State Management Strategy
- When local state is sufficient
- When global state is justified
- Avoiding global store abuse
- Cross-tab and cross-session state

### d) Frontend Observability
- Core Web Vitals (LCP, CLS, INP)
- Error boundaries
- Frontend logging and trace correlation
- Oncall implications of frontend failures

---

## 3. Frontend + Mobile Unification (Your Differentiator)

Use your background to turn this into a strength.

### Key Topics
- Server-Driven UI (SDUI) patterns across web and mobile
- Schema versioning and backward compatibility
- Shared experimentation frameworks (feature flags, A/B tests)
- Unified analytics across platforms
- Rendering vs interaction differences (touch vs click)

**Strong EM framing:**  
> “Frontend and mobile are different renderers of the same product contract.”

---

## 4. Frontend System Design Prompts to Practice

Treat these like backend system design — but **UI-first**.

### Prompt 1: High-Traffic Product Landing Page
Design a landing page that:
- Loads fast on low-end devices
- Supports A/B testing
- Works across web and mobile (WebView)
- Iterates daily without app releases

**Must cover:**
- Rendering strategy (SSR vs CSR vs hybrid)
- Data fetching and caching
- Component ownership
- Rollout and rollback
- Metrics

---

### Prompt 2: Real-Time Feed or Chat UI
Design a frontend system for:
- Real-time updates
- Pagination and infinite scroll
- Offline or flaky networks
- Memory constraints

**Must cover:**
- State consistency
- Backpressure and throttling
- UI virtualization
- Failure recovery

---

## 5. Testing and Release Safety (Frontend-Specific)

Translate your mobile rigor directly to frontend.

### Testing Pyramid
- Unit tests: pure functions, reducers, utilities
- Integration tests: components + data
- E2E tests: critical user flows only

### Release Safety
- Feature flags
- Canary releases
- Kill switches
- Gradual rollout of JavaScript bundles

**Interview signal:**  
Calling out *what you intentionally do not test* (and why) demonstrates senior judgment.

---

## 6. 30 / 60 / 90 Plan (Frontend Lens)

Integrate this into your Week 5 deliverables.

### First 30 Days
- Audit bundle size, LCP, and error rates
- Map UI ownership and dependencies
- Identify duplication and tech debt

### 60 Days
- Define frontend architecture principles
- Clarify component ownership and contracts
- Improve CI stability and build reliability

### 90 Days
- Ship measurable performance wins
- Enable faster experimentation
- Reduce cross-team friction

---

## 7. Daily Prep Plan (Week 5 Only)

### Mon–Fri (45–60 min/day)
- 15 min — Frontend concept deep dive
- 20 min — Analyze a real product page (DevTools walkthrough)
- 15–25 min — Practice one frontend design prompt verbally

### Saturday
- One full frontend system design mock (record yourself)

### Sunday
- Convert one mobile system design story into a frontend-first version

---

## 8. Frontend Interview Talk Track (Default)

Use this structure for **any** frontend-related question:

1. User experience goals
2. Rendering and data strategy
3. Performance considerations
4. Failure modes
5. Team ownership and iteration velocity
6. Metrics and feedback loops
