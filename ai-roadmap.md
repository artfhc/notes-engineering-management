# AI Strategy for Search Team: 3-Month Roadmap

**Team:** 7-person Frontend/Mobile Team (Search)  
**Scope:** Android, iOS, BFF (Java Spring + SDUI with Kotlin DSL)  
**Goal:** Improve developer productivity through AI coding agents  
**Author:** [Your Name]  
**Date:** March 31, 2026

---

## Executive Summary

This proposal outlines a 3-month AI strategy for the Search team, focused on solving our core productivity bottleneck: **context switching across platforms**. Instead of jumping straight to "agents," we will build the **harness first** – documentation, evaluation frameworks, and safety rails – then scale to full agents by Month 3.

**The Problem:** Developers switching between Android, iOS, and BFF tasks lose 20-30 minutes rebuilding mental context. With AI tools, this should be instant.

**The Approach:** Documentation-centric AI adoption using structured SDUI schemas, knowledge graphs, and Claude Code as the primary harness.

**Expected Outcome:** 15-25% improvement in feature cycle time by Month 3, with strong evidence from a Microsoft/MIT study showing **55.8% faster task completion** with AI-assisted coding.

---

## 1. Why "Harness First, Agents Second"?

Industry research from Anthropic and DORA shows a clear failure pattern: teams that jump straight to AI agents without evaluation frameworks see backsliding, technical debt, and developer frustration. Teams that build the harness first (safety, observability, evaluation) scale successfully.

**Our phased approach:**

| Phase | Timeframe | Focus | Deliverable |
|-------|-----------|-------|-------------|
| **Month 1** | Foundation | Documentation + Knowledge Graph | 80% adoption, baseline established |
| **Month 2** | Integration | Evaluation + E2E Testing | Leading metrics trending positive |
| **Month 3** | Scale | Internal Agents + Multi-platform | 15-25% cycle time improvement |

---

## 2. Month 1: Foundation (Knowledge + Documentation)

### 2.1 The Context Problem

When an Android developer needs to pick up iOS work, the friction isn't language syntax – it's understanding:
- Which SDUI component maps to what view structure
- Platform-specific patterns in our codebase  
- Cross-platform API contracts

### 2.2 Solution: Structured Documentation

Instead of building custom MCP servers (higher complexity), we start with **documentation-as-code**:

**Deliverable 1: SDUI Schema Documentation**
```
repository-root/
├── .cursorrules                    # Claude Code reads this automatically
├── docs/
│   ├── sdui-schema/
│   │   ├── components.json         # Component definitions, props
│   │   ├── layouts.json            # Layout rules
│   │   ├── client-mapping.json     # Server DSL → Client view mapping
│   │   └── examples/
│   │       ├── ProductCard.kt      # Kotlin DSL
│   │       └── ProductCard.swift   # iOS equivalent
│   └── platform-guides/
│       ├── android-to-ios.md       # "If you know Android..."
│       └── ios-to-android.md       # "If you know iOS..."
```

**The `.cursorrules` Pattern:**
```yaml
# .cursorrules
context:
  sdui:
    schema_path: ./docs/sdui-schema/
    rules:
      - "All text must use Text component, never raw strings"
      - "Images require both url and fallbackUrl"
      - "SearchResultCard must include price and rating fields"
      
cross_platform:
  android_to_ios:
    RecyclerView: "SwiftUI.List or UIHostingController"
    ViewModel: "ObservableObject with @Published"
  ios_to_android:
    SwiftUI.View: "Composable or View with Binding"
```

**Deliverable 2: Knowledge Graph**

Use `codebase-memory-mcp` to index our four repositories (SDUI DSL, Android, iOS, BFF). Research shows this delivers **20-120x token reduction** vs. file-by-file grep.

**Example queries:**
```
"What SDUI components render on both Android and iOS?"
"Show me all usages of SearchResultCard across platforms"
"What BFF endpoints serve the ProductDetail screen?"
```

**Tool:** `codebase-memory-mcp` (open source, zero dependencies, 66 languages)

### 2.3 Month 1 Success Criteria

| Metric | Target |
|--------|--------|
| Documentation Coverage | 100% of core SDUI components documented |
| Knowledge Graph Index | All 4 repos indexed and queryable |
| Team Adoption | 100% team has tried Claude Code with new context |
| Baseline Established | PR velocity, context-switch time measured |
| Quality | No regressions in code quality (lint/test passing) |

---

## 3. Month 2: Integration (Evaluation + Safety)

### 3.1 The Problem: How Do We Trust AI Output?

Anthropic's research: *"Teams can get surprisingly far through manual testing... but after scaling, building without evals starts to break down."*

We need safety rails **before** we scale.

### 3.2 Solution: Evaluation Framework

**Three Types of Evaluation:**

1. **Capability Evals:** "What can the agent do well?" (start with low pass rate, improve over time)
2. **Regression Evals:** "Does it still do what it used to?" (must be 95%+ pass rate)
3. **Grader Pipeline:** Code-based + LLM-based + human spot-check

**Tool:** Pydantic Evals (code-first, integrates with CI/CD)

### 3.3 E2E Testing Integration (Maestro)

For SDUI specifically, we adopt **Maestro** for black-box testing:
- Tests stay stable when UI structure changes
- YAML-based (no code changes needed in app)
- <1% flakiness (vs. 5-15% for Appium)
- Perfect for validating SDUI server changes

**Example Maestro flow:**
```yaml
# test-search-flow.yaml
appId: com.coupang.search
---
- launchApp
- assertVisible: "Search"
- tapOn:
    id: "search-input"
- inputText: "running shoes"
- tapOn:
    id: "search-submit"
- assertVisible: "Search Results"
# Screenshots captured for visual validation
```

### 3.4 Month 2 Success Criteria

| Metric | Target |
|--------|--------|
| Evals Created | 20+ regression evals, 50+ capability evals |
| Pass Rate | 95%+ on regression suite |
| E2E Integration | Maestro running on critical user journeys |
| Leading Indicators | PR turnaround time, commit frequency trending positive |
| Quality Guard | ktlint/detekt/detekt running on all AI-generated code |

---

## 4. Month 3: Scale (Internal Agents)

### 4.1 The Payoff: Coding Agents + Tooling Agents

With harness in place, we deploy two agent types:

**1. Cross-Platform Coding Agent**
- Understands Android → iOS translation using our documentation
- Generates scaffold code following platform patterns
- Validates changes against our knowledge graph

**2. Internal Tooling Agents**
- SDUI validator agent (checks DSL → client mapping)
- Documentation agent (keeps .cursorrules fresh)
- E2E test generator agent (creates Maestro flows from PRs)

### 4.2 Measurement: Proving ROI

**Primary Metrics (vs. Month 1 baseline):**

| Metric | Target | Measurement |
|--------|--------|-------------|
| Feature Cycle Time | -15-25% | Jira ticket analysis (design → prod) |
| Cross-Platform Task Time | -30% | Self-reported + git analysis |
| PR Merge Time | -15% | GitHub PR data |
| Context Switches/Week | -25% | Self-reported friction score |
| Developer Satisfaction | +0.5 points | Weekly 5-point survey |

**Validation from Industry:**
- Microsoft/MIT controlled study: **55.8% faster task completion** with AI coding tools
- Google/DORA finding: AI acts as **amplifier** – magnifies team strengths

### 4.3 Month 3 Success Criteria

| Metric | Target |
|--------|--------|
| Cycle Time Improvement | 15-25% vs. baseline |
| Quality | No increase in production incidents |
| Adoption | 80%+ of features use AI assistance |
| Satisfaction | >4.0/5 developer satisfaction |
| Case Studies | 2+ documented wins for leadership |

---

## 5. Tool Stack Summary

| Layer | Tool | Purpose | Cost |
|-------|------|---------|------|
| **Agent Core** | Claude Code | Primary dev environment | Existing |
| **Documentation** | `.cursorrules` + Markdown | Structured SDUI context | Free |
| **Knowledge Graph** | `codebase-memory-mcp` | Cross-platform code relationships | Free |
| **Evaluation** | Pydantic Evals | Systematic testing (Phase 2) | Free |
| **E2E Testing** | Maestro | SDUI validation | Free (open source) |
| **Static Analysis** | ktlint/detekt/detekt | Code quality gates | Free |
| **CI/CD** | GitHub Actions | Automation | Existing |

**Total Additional Cost:** $0 (we already use Claude Code)

---

## 6. Resource Allocation

**This is a 7-person team. We keep it lean:**

| Phase | Team Commitment |
|-------|-----------------|
| Month 1 | 2-3 days/week documentation sprint (rotating) |
| Month 2 | 1 engineer leading evaluation setup (1-2 days/week) |
| Month 3 | Full team on agent features (adoption time) |

**Who does what:**
- **Tech Lead** (you): Evaluation framework, leadership presentation
- **Android Lead:** Knowledge graph for Android + BFF
- **iOS Lead:** Cross-platform documentation
- **SDUI Owner:** Schema documentation + Maestro flows
- **Team:** Rotating documentation duty (2-3 days/person/Month 1)

---

## 7. Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Documentation becomes stale | Weekly "documentation debt" time in sprints |
| AI generates bad code | ktlint/detekt/detekt automated gates |
| Team adoption lags | Start with willing volunteers, share wins |
| Knowledge graph doesn't help | Start with SDUI repo only, expand if useful |
| Metrics don't improve | Focus on leading indicators (adoption → outcomes) |

**Key Insight from Research:** Only 16.3% of devs report AI as "much more productive." Success requires **workflow integration**, not just buying tools. We're investing in integration (Month 1-2) before expecting outcomes (Month 3).

---

## 8. Success Narrative for Leadership

**Month 1 Pitch:** "We've built the foundation – every SDUI component is now documented and queryable across platforms. We're measuring baseline before expecting gains."

**Month 2 Pitch:** "Safety rails are in place. We can now trust AI-generated code because it passes automated evaluation + E2E tests. Early signals are positive."

**Month 3 Pitch:** "15-25% faster feature delivery. Context switching across Android/iOS is now measured in minutes, not hours. Here's the data."

**Anchor Statistic:** Microsoft/MIT study shows **55.8% faster task completion** with AI coding tools. We're targeting 15-25% – a conservative, achievable number.

---

## 9. Next Steps

**This Week:**
1. [ ] Approve roadmap approach
2. [ ] Assign Month 1 documentation lead (Android + iOS + SDUI owners)
3. [ ] Schedule 2-day documentation sprint
4. [ ] Set up `codebase-memory-mcp` on test machine

**Month 1:**
- Document top 20 SDUI components
- Index all 4 repos
- Create `.cursorrules` with cross-platform mappings
- Measure baseline (PR cycle time, context-switch surveys)

---

## Appendix: Research Sources

1. **Anthropic Research:** "Demystifying Evals for AI Agents" – [anthropic.com/engineering](https://anthropic.com/engineering)
2. **Microsoft/MIT Study:** "The Impact of AI on Developer Productivity" (arXiv 2302.06590) – **55.8% faster**
3. **Google/DORA 2025:** State of AI-assisted Dev – AI acts as **amplifier**
4. **Cerbos Study (Sept 2025):** Only **16.3%** report "much more productive" – highlights need for workflow integration

**Tools Referenced:**
- `codebase-memory-mcp`: https://github.com/kamilstan/codebase-memory-mcp
- Pydantic Evals: https://ai.pydantic.dev/evals/
- Maestro: https://maestro.dev
- `.cursorrules`: Cursor/Claude Code context files

---

*This roadmap focuses on practical, low-cost wins that compound over time. The "harness first" approach ensures we can scale safely, while the documentation-centric strategy delivers immediate value without custom infrastructure.*
