# Core Flow: Deterministic Memory Capture with Claude Code Hooks

The clean way to enforce memory with hooks is to stop thinking of memory as "whatever Claude happens to remember" and instead build a deterministic capture pipeline around Claude Code's lifecycle. Hooks run at specific lifecycle points, receive JSON on stdin, and can block, allow, or inject context depending on the event. Anthropic explicitly positions them as a way to make actions happen reliably, rather than hoping the model chooses to do them.

## The Core Idea

Use three memory layers:

| Layer | Purpose | Source of truth | Check in? |
|-------|---------|-----------------|-----------|
| `CLAUDE.md` | durable team rules and standards | human-reviewed | yes |
| `.claude/memory-candidate/*.jsonl` | raw learnings captured automatically | hook-generated | usually no |
| `.claude/team-memory/*.md` | curated summaries promoted from candidates | human-reviewed or CI-reviewed | yes |

This matches Anthropic's distinction between CLAUDE.md for instructions you write and auto memory for learnings Claude writes for itself. Anthropic also notes both are just context, not hard enforcement, which is why hooks are useful for making the update process deterministic.

## The Strategy

### 1. Capture candidate memories automatically

Do not write directly into CLAUDE.md from a hook.

Instead, use hooks to append structured records into a candidate log such as:

```
.claude/memory-candidate/
  tool-use.jsonl
  prompts.jsonl
  stop-events.jsonl
  decisions.jsonl
```

Why: Claude Code memory is loaded as context every session, and Anthropic recommends keeping CLAUDE.md concise, around under 200 lines per file. If you auto-append everything into checked-in memory, it will rot fast.

### 2. Promote only stable patterns

Promote a candidate into checked-in memory only if it is:

- repeated
- repo-relevant
- likely useful next session
- not a secret
- not tied to one person's machine

Anthropic's memory guidance says CLAUDE.md should contain what you would otherwise need to re-explain in every session: build commands, conventions, architecture, common workflows.

### 3. Enforce memory generation with hooks

Use hooks at these points:

- **UserPromptSubmit**: capture explicit "remember this" instructions and inject reminders into context when needed. Anthropic says UserPromptSubmit hooks receive the prompt text, and anything they write to stdout is added to Claude's context.
- **PostToolUse**: capture concrete evidence after edits, writes, and bash commands. Anthropic documents PostToolUse as firing after tool execution and shows examples logging bash commands.
- **Stop**: force a final summarization pass before Claude stops. Anthropic documents Stop hooks and supports both prompt-based and agent-based verification here.
- **PreToolUse**: block direct edits to canonical checked-in memory unless they go through your promotion path. Anthropic says PreToolUse hooks can deny actions and those denials apply even in bypass permission mode.

## The Workflow

### A. Raw Collection

Every useful event becomes a small structured record.

Example record:

```json
{
  "timestamp": "2026-04-15T21:05:33Z",
  "event": "PostToolUse",
  "category": "debugging_insight",
  "candidate_key": "android_test_requires_redis",
  "statement": "API tests require a local Redis instance before running integration tests.",
  "evidence": {
    "tool": "Bash",
    "command": "./gradlew integrationTest",
    "exit_code": 1
  },
  "scope": "project",
  "confidence": "medium",
  "promote_after": 2,
  "source_branch": "feature/login-fix"
}
```

### B. Reduction

A reducer script merges duplicates and scores them:

- repeated 2+ times → stronger candidate
- appears across multiple branches/worktrees → stronger candidate
- personal path like `/Users/arthur/...` → discard
- contains token-looking strings → discard

### C. Promotion

A promotion job writes to one of:

- `CLAUDE.md` for durable repo-wide rules
- `.claude/rules/android.md` for scoped rules
- `docs/runbooks/*.md` for procedures
- `docs/adr/*.md` for architectural decisions

Anthropic supports path-scoped rules in `.claude/rules/`, which is better than stuffing everything into one global file.

## The Most Useful Hook Patterns

### 1. PreToolUse: protect canonical memory

Block direct edits to checked-in memory files except through a promotion script.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/protect-memory.sh"
          }
        ]
      }
    ]
  }
}
```

`protect-memory.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

INPUT="$(cat)"
FILE_PATH="$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')"

case "$FILE_PATH" in
  *"/CLAUDE.md"|*"/.claude/rules/"*|*"/.claude/team-memory/"*)
    echo '{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny","permissionDecisionReason":"Do not edit checked-in memory directly. Write candidates to .claude/memory-candidate/ and run promote-memory."}}'
    ;;
  *)
    exit 0
    ;;
esac
```

This works because PreToolUse hooks can deny tool calls with structured JSON, and hook denials can tighten policy even when permission prompts are bypassed.

### 2. UserPromptSubmit: capture explicit memory requests

Capture phrases like "remember that…", "always use…", "for this repo…".

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/capture-prompt-memory.sh"
          }
        ]
      }
    ]
  }
}
```

`capture-prompt-memory.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

INPUT="$(cat)"
PROMPT="$(echo "$INPUT" | jq -r '.prompt // ""')"
OUTDIR="$CLAUDE_PROJECT_DIR/.claude/memory-candidate"
mkdir -p "$OUTDIR"

if echo "$PROMPT" | grep -Eiq '\b(remember|always use|for this repo|team rule|convention|policy)\b'; then
  jq -n \
    --arg ts "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
    --arg event "UserPromptSubmit" \
    --arg statement "$PROMPT" \
    '{
      timestamp: $ts,
      event: $event,
      category: "user_declared_memory",
      statement: $statement,
      scope: "project",
      confidence: "medium",
      promote_after: 1
    }' >> "$OUTDIR/prompts.jsonl"
fi

exit 0
```

This is a good fit because UserPromptSubmit receives the prompt text, and stdout from this event can also inject extra context when needed.

### 3. PostToolUse: collect evidence-backed learnings

This is the most important one.

Capture:

- successful commands worth reusing
- failed commands with known fixes
- edited files that imply ownership or architecture patterns
- repeated formatting or lint fixes

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash|Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/capture-tool-memory.sh"
          }
        ]
      }
    ]
  }
}
```

`capture-tool-memory.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

INPUT="$(cat)"
OUTDIR="$CLAUDE_PROJECT_DIR/.claude/memory-candidate"
mkdir -p "$OUTDIR"

TOOL="$(echo "$INPUT" | jq -r '.tool_name // ""')"

case "$TOOL" in
  Bash)
    CMD="$(echo "$INPUT" | jq -r '.tool_input.command // ""')"
    jq -n \
      --arg ts "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
      --arg tool "$TOOL" \
      --arg cmd "$CMD" \
      '{
        timestamp: $ts,
        event: "PostToolUse",
        category: "command_observation",
        tool: $tool,
        statement: $cmd
      }' >> "$OUTDIR/tool-use.jsonl"
    ;;
  Edit|Write|MultiEdit)
    FILE="$(echo "$INPUT" | jq -r '.tool_input.file_path // ""')"
    jq -n \
      --arg ts "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
      --arg tool "$TOOL" \
      --arg file "$FILE" \
      '{
        timestamp: $ts,
        event: "PostToolUse",
        category: "file_change",
        tool: $tool,
        file: $file
      }' >> "$OUTDIR/tool-use.jsonl"
    ;;
esac

exit 0
```

Anthropic shows PostToolUse hooks as a standard way to log tool activity and also supports HTTP hooks if you want to send these records to a collector service instead of local files.

### 4. Stop: require a memory summary before Claude finishes

This is where you create a candidate summary for check-in.

Use either:

- a prompt hook for cheap summarization
- an agent hook when the hook needs to read changed files, test output, or git diff

Anthropic documents both on Stop, and agent hooks can inspect files and use tools before returning.

Example Stop hook:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "Read .claude/memory-candidate/*.jsonl and the git diff. Produce a concise JSON object with durable candidate memories under keys: facts, rules, recipes, decisions. Exclude secrets, one-off notes, local-machine details, and branch-specific noise. Write only candidates suitable for later check-in. $ARGUMENTS",
            "timeout": 90
          }
        ]
      }
    ]
  }
}
```

Then have that hook write to:

```
.claude/team-memory/pending/YYYY-MM-DD-session-summary.json
```

Important: Anthropic notes Stop hooks can accidentally loop forever unless they check `stop_hook_active`.

## What Should Get Checked In

Only check in material that survives this filter:

| Check in | Do not check in |
|----------|-----------------|
| stable build/test commands | personal preferences |
| recurring debugging gotchas | one-off branch notes |
| architecture boundaries | flaky workaround that is already fixed |
| naming or review conventions | secrets, tokens, URLs for private sandboxes |
| release and rollback workflows | machine-specific paths |

This is aligned with Anthropic's guidance that CLAUDE.md is for standards, workflows, and architecture rather than transient or personal state.

## Best Promotion Targets

Do not promote everything into CLAUDE.md.

Use this mapping:

| Candidate type | Destination |
|---------------|-------------|
| repo-wide rule | `CLAUDE.md` |
| path-specific rule | `.claude/rules/<area>.md` |
| repeatable operational steps | `docs/runbooks/*.md` |
| architectural rationale | `docs/adr/*.md` |
| temporary pending notes | `.claude/team-memory/pending/` |

Anthropic specifically recommends path-scoped rules for codebase-specific sections and says large projects should break instructions into topic-specific rules rather than one big memory file.

## A Good Repo Layout

```
repo/
├─ CLAUDE.md
├─ .claude/
│  ├─ hooks/
│  │  ├─ protect-memory.sh
│  │  ├─ capture-prompt-memory.sh
│  │  ├─ capture-tool-memory.sh
│  │  └─ promote-memory.sh
│  ├─ rules/
│  │  ├─ android.md
│  │  ├─ backend.md
│  │  └─ infra.md
│  ├─ memory-candidate/
│  │  ├─ prompts.jsonl
│  │  ├─ tool-use.jsonl
│  │  └─ decisions.jsonl
│  └─ team-memory/
│     └─ pending/
├─ docs/
│  ├─ runbooks/
│  └─ adr/
```

## My Strongest Recommendation

Use a two-phase commit model for memory:

**Phase 1: hooks collect**

Hooks create structured candidate memory automatically from prompts, tool use, and session stop events. Hooks are ideal here because they are deterministic and can block or inject context based on event type.

**Phase 2: promotion job curates**

A script or CI job converts repeated, high-confidence candidates into checked-in docs:

- `CLAUDE.md`
- `.claude/rules/*.md`
- runbooks
- ADRs

That gives you:

- automatic capture
- auditable diffs
- low memory rot
- safe check-ins

## A Very Practical Promotion Policy

I would use this exact policy:

- promote to CLAUDE.md after the same rule appears 2 times across sessions
- promote to runbook after the same repair workflow appears 2 times
- promote to ADR after a decision candidate has both a "why" and impacted files or systems
- expire unpromoted candidates after 30 days
- require PR review for all checked-in memory changes

## One More Useful Trick

At session start, inject a short generated summary of pending team memory candidates into Claude's context so it can benefit before they are promoted.

That works because SessionStart and UserPromptSubmit hooks can add text to Claude's context via stdout.

Example injected context:

```
Pending team memory candidates:
- Integration tests require local Redis.
- Payments service edits must include rollback notes.
- Android build often needs ./gradlew clean after dependency version changes.
Treat these as provisional until confirmed.
```

That gives you soft learning immediately, without polluting canonical memory too early.

## Bottom Line

The enforcement model is:

1. Protect canonical memory with PreToolUse
2. Capture candidate memory with UserPromptSubmit and PostToolUse
3. Summarize before exit with Stop
4. Promote only reviewed, repeated patterns into checked-in memory

That is the cleanest way to get automatic memory updates that your team can actually commit and trust.
