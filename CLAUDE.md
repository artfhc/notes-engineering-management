# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a personal knowledge repository containing engineering management notes, interview preparation materials, and system design documentation. It contains markdown documents organized into problem areas and system design topics.

## Repository Structure

### Root-Level Documents
- `problem-*.md`: Engineering management problem-solving frameworks and actionable plans
  - `problem-engineer-alignment.md`: Management action plan for addressing team feedback (project clarity, quality gates, process improvements)
  - `problem-interview-prep.md`: 8-week EM interview preparation roadmap
  - `problem-interview-prep-fe.md`: Frontend-specific EM interview preparation
  - `problem-interview-prep-fe-where.md`: Frontend interview topics (companies like Meta, Google, Discord, Anthropic)
  - `problem-fridge-project-deepdive.md`: Production estimate for hybrid web-in-app project (WebView architecture)
  - `problem-fridge-project-deepdive-2.md`: Additional project analysis
  - `problem-fridge-project-deepdive-q&a.md`: Q&A on the fridge project

### System Design Directory
- `system-design/`: Contains system design interview preparation materials
  - `drawing-app.md`: Collaborative drawing application design (operation-based sync, realtime collaboration, offline support)

## Document Conventions

### Problem Documents Format
Problem documents follow a structured approach:
1. **Problem framing**: Clear articulation of the issue
2. **Root cause analysis**: Management or systemic perspectives
3. **Action plans**: Concrete, EM-owned solutions
4. **Expected outcomes**: Measurable results

### System Design Documents Format
System design documents typically include:
1. **Scope clarification**: MVP vs non-goals
2. **Data model**: Core entities and relationships
3. **API surface**: Clean, interview-friendly endpoints
4. **Architecture**: Services, storage, scaling considerations
5. **Reliability & edge cases**: Failure modes, conflict handling
6. **Performance considerations**: Latency, caching, optimization

### Interview Prep Documents Format
Interview preparation materials include:
1. **Weekly roadmaps**: Structured learning paths
2. **Practice frameworks**: Reusable answer templates (STAR, system design talk tracks)
3. **Company-specific guidance**: Tailored approaches for different orgs
4. **Deliverables**: Concrete outputs for each phase

## Key Patterns and Principles

### Management Philosophy (from problem-engineer-alignment.md)
- Ambiguity is more expensive than slow decisions
- Starting fast is worse than starting clear
- Quality failures are management problems before they are engineering problems
- Repeated issues indicate missing system-level guardrails

### System Design Approach (from drawing-app.md)
- Operation-based collaboration systems with server-ordered revisions
- Snapshot + op-log storage for history and recovery
- Offline-first design with local state + sync reconciliation
- Explicit handling of conflict resolution (last-write-wins or CRDT)

### Production Estimation Approach (from fridge-project documents)
- Clear MVP vs out-of-scope boundaries
- Risk identification with concrete mitigations
- Week-by-week milestones
- Hybrid architecture considerations (WebView + native shell)
- Performance budgets and monitoring requirements

## Working with This Repository

### When Adding New Documents
1. Use descriptive filenames with `problem-` prefix for management/execution topics
2. Place system design materials in `system-design/` directory
3. Follow the established document structure for consistency
4. Include concrete examples and metrics where applicable

### When Editing Existing Documents
1. Preserve the document structure (headers, numbered lists, tables)
2. Maintain the action-oriented tone for problem documents
3. Keep technical depth appropriate for EM-level interviews (architecture over implementation)
4. Ensure examples are realistic and drawn from production contexts

### Git Workflow
This is a personal repository with a single branch (`main`). Commits should:
- Use descriptive commit messages that explain "why" not just "what"
- Group related changes together
- Follow the existing commit style (see recent commits for examples)

## Content Philosophy

### What Belongs Here
- Engineering management frameworks and action plans
- Interview preparation materials (behavioral, system design, coding)
- Production project estimation and risk analysis
- System architecture decisions and trade-offs

### What Doesn't Belong Here
- Actual code implementations (this is a notes repository)
- Company-confidential information
- Generic advice without actionable frameworks
- Overly detailed step-by-step tutorials