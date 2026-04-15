# Detail Flow: Memory Promotion Strategies

Promotion is NOT automatic by default. It only happens if you design it. Otherwise, Claude just writes/uses memory opportunistically.

## Two Ways Promotion Can Work

### 1. "Claude decides" (weak, not reliable)

You prompt Claude like: "If something is important, add it to memory"

What actually happens:

- Sometimes it writes to a file
- Sometimes it forgets
- No consistency
- No safety (can overwrite or add junk)

This depends on LLM judgment — not deterministic.

**Verdict:** good for experiments, bad for teams

### 2. Hook + pipeline driven (what you want)

Promotion becomes a controlled system, not a suggestion.

**Claude proposes, system decides.**

## Promotion Pipeline (the Real Model)

Think of it like a CI pipeline for memory:

```
[Claude session]
      ↓
[Hooks collect candidates]
      ↓
[Candidate store (.jsonl)]
      ↓
[Reducer / scorer]
      ↓
[Promotion decision]
      ↓
[PR or auto-commit]
```

## Three Promotion Strategies

### 1. Fully Automatic (high leverage, needs guardrails)

**How it works:**

- Hook collects candidates
- Script evaluates rules
- If threshold met → auto-write to memory files

**Example rule:**

```yaml
promote_if:
  occurrences >= 2
  AND confidence >= medium
  AND no_secrets == true
```

**Example outcome:**

```diff
+ ## Build Notes
+ - Integration tests require local Redis running
```

**Pros:** Zero human effort, scales well

**Cons:** Risk of memory rot / garbage, needs strong filters

### 2. PR-based Promotion (best for teams)

**How it works:**

- Hook generates candidate summary
- CI job creates PR like:

```diff
docs/runbooks/testing.md

+ ## Integration Tests
+ - Requires local Redis before running
+ - Command: docker-compose up redis
```

- Human reviews & merges

**Pros:** High quality, auditable, fits your engineering culture (like code review)

**Cons:** Slight friction

**This is what I recommend for your team.**

### 3. Interactive Promotion (Claude asks)

Claude says: "This seems important. Should I add it to team memory?"

**Pros:** Safe, simple

**Cons:** Interrupts flow, engineers ignore it over time

## The Key Design Principle

Claude should NEVER directly modify canonical memory.

Instead:

| Role | Responsibility |
|------|---------------|
| Claude | propose candidates |
| Hooks | enforce capture |
| Scripts/CI | decide promotion |
| Humans | approve (optional but recommended) |

## What Determines Promotion?

You define promotion signals, not Claude.

**Strong signals (use these):**

- repeated occurrence (≥2 times)
- appears across different sessions / branches
- tied to build/test failures
- affects multiple files/services

**Weak signals (ignore):**

- one-off fix
- stylistic preference
- personal workflow
- temporary workaround

## Example Scoring System

```json
{
  "candidate": "tests require redis",
  "signals": {
    "occurrences": 3,
    "distinct_sessions": 2,
    "failure_related": true,
    "scope": "project"
  },
  "score": 0.87,
  "decision": "promote"
}
```

## Guardrails (VERY important)

Without these, your system will degrade fast.

### 1. Block Direct Edits

Use PreToolUse hook:

- deny writes to `CLAUDE.md`
- deny writes to `/docs/runbooks`

### 2. Filter Bad Memory

Before promotion:

- remove secrets (tokens, keys)
- remove local paths (`/Users/...`)
- remove timestamps / noise
- deduplicate

### 3. TTL for Candidates

```
expire_after: 30 days
```

### 4. Size Limits

- `CLAUDE.md` stays small (<200 lines)
- push everything else to `rules/`, `runbooks/`, ADRs

## What This Looks Like in Practice

**Day 1:** Claude fails tests → learns "Redis required" → stored as candidate

**Day 2:** Same issue again → candidate score increases

**Day 3 (CI job):** Promotion triggers → PR created:

```diff
docs/runbooks/testing.md

+ ## Integration Tests
+ Requires Redis running locally
```

**Day 4:** Merged → now all engineers benefit, all Claude sessions benefit

## The "Real Hack"

Don't ask Claude to manage memory. Build a system where memory is a byproduct of work.

## Recommendation

Given your engineering manager role, emphasis on quality / reliability, and interest in ML-driven systems:

Go with PR-based promotion + automatic candidate capture:

1. Hooks → collect everything
2. Reducer → score
3. CI → generate PR
4. Team → review like code

## Simple Mental Model

| Role | Analogy |
|------|---------|
| Claude | junior engineer writing notes |
| Hooks | logging system |
| Reducer | signal detector |
| CI | tech lead |
| PR review | team judgment |
