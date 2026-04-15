# Update Flow: Memory Scoring and Auto-PR Creation

A ready-to-use starter setup for memory scoring + auto-PR creation.

It assumes:

- hooks write candidate memories into `.claude/memory-candidate/*.jsonl`
- you want to promote repeated, durable items
- promotion should happen through a reviewable PR, not direct edits to main

Claude Code's built-in auto memory is on by default in current versions, but it is still just context Claude saves for itself; for team memory you'll get a much more reliable result by treating candidate memories as inputs to a deterministic pipeline and then checking in curated docs. Anthropic documents both CLAUDE.md/rules and auto memory as project context mechanisms, and hooks as the mechanism for deterministic workflow automation.

## Repo Layout

```
.claude/
  memory-candidate/
    prompts.jsonl
    tool-use.jsonl
    decisions.jsonl
  team-memory/
    pending/
  scripts/
    score_memory.py
    render_memory_docs.py
.github/
  workflows/
    memory-promotion.yml
CLAUDE.md
docs/
  runbooks/
  adr/
```

## 1) Scoring Script

Save as: `.claude/scripts/score_memory.py`

```python
#!/usr/bin/env python3
import json
import re
import sys
from pathlib import Path
from collections import defaultdict
from datetime import datetime, timezone, timedelta

ROOT = Path(__file__).resolve().parents[2]
CANDIDATE_DIR = ROOT / ".claude" / "memory-candidate"
OUTPUT_DIR = ROOT / ".claude" / "team-memory" / "pending"

MAX_AGE_DAYS = 30
PROMOTE_THRESHOLD = 3.0

TOKEN_PATTERNS = [
    r"ghp_[A-Za-z0-9_]{20,}",
    r"github_pat_[A-Za-z0-9_]{20,}",
    r"sk-[A-Za-z0-9]{20,}",
    r"AKIA[0-9A-Z]{16}",
]

PATH_PATTERNS = [
    r"/Users/[^/\s]+/",
    r"/home/[^/\s]+/",
    r"[A-Z]:\\Users\\[^\\\s]+\\",
]

NOISE_PATTERNS = [
    r"\btmp\b",
    r"\btemporary\b",
    r"\bone-off\b",
]

CATEGORY_TO_DEST = {
    "rule": "claude_md",
    "fact": "claude_md",
    "recipe": "runbook",
    "decision": "adr",
    "debugging_insight": "runbook",
    "command_observation": "runbook",
    "user_declared_memory": "claude_md",
}

def now_utc():
    return datetime.now(timezone.utc)

def parse_ts(ts: str):
    try:
        return datetime.fromisoformat(ts.replace("Z", "+00:00"))
    except Exception:
        return None

def normalize_statement(text: str) -> str:
    text = text.strip()
    text = re.sub(r"`[^`]+`", "", text)
    text = re.sub(r"\s+", " ", text)
    text = text.lower()
    text = re.sub(r"[^a-z0-9\s:/._-]", "", text)
    return text

def looks_secret(text: str) -> bool:
    return any(re.search(p, text) for p in TOKEN_PATTERNS)

def looks_machine_specific(text: str) -> bool:
    return any(re.search(p, text) for p in PATH_PATTERNS)

def looks_noise(text: str) -> bool:
    return any(re.search(p, text, re.I) for p in NOISE_PATTERNS)

def classify_category(raw: dict) -> str:
    cat = raw.get("category", "").strip()
    if cat:
        return cat
    event = raw.get("event", "")
    stmt = raw.get("statement", "")
    if "decision" in stmt.lower():
        return "decision"
    if event == "UserPromptSubmit":
        return "user_declared_memory"
    return "fact"

def score_candidate(records: list[dict]) -> float:
    score = 0.0
    occurrences = len(records)
    distinct_sessions = len({r.get("session_id", r.get("source_branch", "unknown")) for r in records})
    distinct_branches = len({r.get("source_branch", "unknown") for r in records})

    score += min(occurrences, 5) * 0.8
    score += min(distinct_sessions, 3) * 0.7
    score += min(distinct_branches, 2) * 0.4

    if any(r.get("failure_related") for r in records):
        score += 0.8
    if any(r.get("confidence") == "high" for r in records):
        score += 0.6
    if any(r.get("scope") == "project" for r in records):
        score += 0.5

    return round(score, 2)

def main():
    OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

    records_by_key = defaultdict(list)
    cutoff = now_utc() - timedelta(days=MAX_AGE_DAYS)

    for file in sorted(CANDIDATE_DIR.glob("*.jsonl")):
        for line_num, line in enumerate(file.read_text(encoding="utf-8").splitlines(), start=1):
            if not line.strip():
                continue
            try:
                raw = json.loads(line)
            except json.JSONDecodeError:
                continue

            statement = (raw.get("statement") or "").strip()
            if not statement:
                continue

            ts = parse_ts(raw.get("timestamp", ""))
            if ts and ts < cutoff:
                continue

            if looks_secret(statement) or looks_machine_specific(statement) or looks_noise(statement):
                continue

            normalized = normalize_statement(statement)
            if len(normalized) < 12:
                continue

            category = classify_category(raw)
            key = raw.get("candidate_key") or normalized

            records_by_key[(category, key)].append({
                **raw,
                "normalized_statement": normalized,
                "source_file": str(file),
                "source_line": line_num,
            })

    promoted = []
    rejected = []

    for (category, key), records in records_by_key.items():
        score = score_candidate(records)
        best = sorted(records, key=lambda r: (
            r.get("confidence") == "high",
            r.get("scope") == "project",
            len(r.get("statement", "")),
        ), reverse=True)[0]

        candidate = {
            "candidate_key": key,
            "category": category,
            "destination": CATEGORY_TO_DEST.get(category, "claude_md"),
            "statement": best.get("statement", "").strip(),
            "score": score,
            "occurrences": len(records),
            "source_branches": sorted({r.get("source_branch", "unknown") for r in records}),
            "sources": [
                {
                    "file": r["source_file"],
                    "line": r["source_line"],
                    "timestamp": r.get("timestamp"),
                }
                for r in records[:10]
            ],
        }

        if score >= PROMOTE_THRESHOLD:
            promoted.append(candidate)
        else:
            rejected.append(candidate)

    promoted.sort(key=lambda x: (-x["score"], x["category"], x["statement"]))
    rejected.sort(key=lambda x: (-x["score"], x["category"], x["statement"]))

    out = {
        "generated_at": now_utc().isoformat(),
        "promoted": promoted,
        "rejected": rejected,
        "threshold": PROMOTE_THRESHOLD,
        "max_age_days": MAX_AGE_DAYS,
    }

    out_file = OUTPUT_DIR / "memory-score.json"
    out_file.write_text(json.dumps(out, indent=2), encoding="utf-8")
    print(f"Wrote {out_file} with {len(promoted)} promoted candidates.")

if __name__ == "__main__":
    main()
```

## 2) Render Promoted Memory into Checked-in Docs

Save as: `.claude/scripts/render_memory_docs.py`

```python
#!/usr/bin/env python3
import json
from pathlib import Path
from collections import defaultdict
from datetime import datetime, timezone

ROOT = Path(__file__).resolve().parents[2]
SCORE_FILE = ROOT / ".claude" / "team-memory" / "pending" / "memory-score.json"

CLAUDE_MD = ROOT / "CLAUDE.md"
RUNBOOK = ROOT / "docs" / "runbooks" / "team-memory.md"
ADR = ROOT / "docs" / "adr" / "team-memory-decisions.md"

HEADER = (
    "<!-- Generated by memory promotion workflow. Edit source candidates or promotion logic, "
    "not this block manually. -->\n"
)

def ensure_parent(path: Path):
    path.parent.mkdir(parents=True, exist_ok=True)

def load_score():
    return json.loads(SCORE_FILE.read_text(encoding="utf-8"))

def upsert_section(path: Path, section_title: str, lines: list[str]):
    ensure_parent(path)
    existing = path.read_text(encoding="utf-8") if path.exists() else ""
    marker_start = f"<!-- START:{section_title} -->"
    marker_end = f"<!-- END:{section_title} -->"
    block = (
        f"{marker_start}\n"
        f"## {section_title}\n\n" +
        ("\n".join(lines).rstrip() + "\n" if lines else "_None._\n") +
        f"{marker_end}\n"
    )

    if marker_start in existing and marker_end in existing:
        import re
        pattern = re.compile(
            re.escape(marker_start) + r".*?" + re.escape(marker_end),
            flags=re.DOTALL
        )
        updated = pattern.sub(block.strip(), existing)
        path.write_text(updated + ("\n" if not updated.endswith("\n") else ""), encoding="utf-8")
    else:
        prefix = existing.rstrip() + ("\n\n" if existing.strip() else "")
        path.write_text(HEADER + prefix + block, encoding="utf-8")

def main():
    data = load_score()
    promoted = data.get("promoted", [])

    groups = defaultdict(list)
    for item in promoted:
        groups[item["destination"]].append(item)

    claude_lines = []
    runbook_lines = []
    adr_lines = []

    for item in groups.get("claude_md", []):
        claude_lines.append(f"- {item['statement']}  <!-- score:{item['score']} occ:{item['occurrences']} -->")

    for item in groups.get("runbook", []):
        runbook_lines.append(f"- {item['statement']}  <!-- score:{item['score']} occ:{item['occurrences']} -->")

    for item in groups.get("adr", []):
        adr_lines.append(f"- {item['statement']}  <!-- score:{item['score']} occ:{item['occurrences']} -->")

    upsert_section(CLAUDE_MD, "Promoted Team Memory", claude_lines)
    upsert_section(RUNBOOK, "Promoted Operational Memory", runbook_lines)
    upsert_section(ADR, "Promoted Decisions", adr_lines)

    summary = ROOT / ".claude" / "team-memory" / "pending" / "promotion-summary.md"
    summary.parent.mkdir(parents=True, exist_ok=True)
    summary.write_text(
        "# Memory Promotion Summary\n\n"
        f"- Generated at: {datetime.now(timezone.utc).isoformat()}\n"
        f"- Promoted candidates: {len(promoted)}\n"
        f"- Threshold: {data.get('threshold')}\n",
        encoding="utf-8"
    )
    print("Rendered promoted memory docs.")

if __name__ == "__main__":
    main()
```

## 3) GitHub Action to Score + Create PR

Save as: `.github/workflows/memory-promotion.yml`

```yaml
name: Memory Promotion

on:
  workflow_dispatch:
  schedule:
    - cron: "17 14 * * 1-5" # weekdays, 14:17 UTC

permissions:
  contents: write
  pull-requests: write

jobs:
  promote-memory:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Score memory candidates
        run: python .claude/scripts/score_memory.py

      - name: Render promoted docs
        run: python .claude/scripts/render_memory_docs.py

      - name: Detect changes
        id: changes
        run: |
          if git diff --quiet; then
            echo "has_changes=false" >> "$GITHUB_OUTPUT"
          else
            echo "has_changes=true" >> "$GITHUB_OUTPUT"
          fi

      - name: Create pull request
        if: steps.changes.outputs.has_changes == 'true'
        uses: peter-evans/create-pull-request@v7
        with:
          commit-message: "docs(memory): promote team memory candidates"
          branch: "automation/memory-promotion"
          delete-branch: true
          title: "docs(memory): promote team memory candidates"
          body: |
            This PR was generated automatically from memory candidates collected by Claude hooks.

            Included:
            - refreshed scored candidates
            - promoted durable items into checked-in docs
            - updated pending promotion summary

            Review checklist:
            - [ ] No secrets or machine-specific details
            - [ ] Items are durable and team-relevant
            - [ ] Promotions belong in the chosen destination files
          labels: |
            automation
            docs
            ai-memory
          reviewers: |
            your-github-handle
          draft: true
```

The `create-pull-request` action is designed for exactly this workflow: modify files in the Actions workspace, commit them to a branch, and open a PR automatically.

## 4) Candidate JSONL Format

Your hooks should append records that look like this:

```jsonl
{"timestamp":"2026-04-15T21:05:33Z","event":"UserPromptSubmit","category":"user_declared_memory","statement":"For this repo, always run Android unit tests before proposing a fix.","scope":"project","confidence":"medium","session_id":"abc123","source_branch":"main"}
{"timestamp":"2026-04-15T21:06:11Z","event":"PostToolUse","category":"command_observation","statement":"Integration tests require local Redis running first.","scope":"project","confidence":"high","failure_related":true,"session_id":"abc123","source_branch":"main"}
{"timestamp":"2026-04-16T09:12:02Z","event":"PostToolUse","category":"command_observation","statement":"Integration tests require local Redis running first.","scope":"project","confidence":"high","failure_related":true,"session_id":"def456","source_branch":"feature/payments"}
```

With the default scoring above, repeated cross-session items like the Redis example will get promoted.

## 5) Promotion Logic in Plain English

This setup promotes when a candidate accumulates enough signal:

- repeated occurrences
- multiple sessions
- multiple branches
- failure-related evidence
- project scope
- optionally high confidence

It rejects or filters:

- secrets and token-like strings
- machine-specific paths
- very short or noisy statements
- stale candidates older than 30 days

That fits the safest model for automation: GitHub warns that letting workflows create PRs can be a security risk if those PRs are merged without oversight, so the workflow above opens a draft PR for review rather than merging automatically. GitHub also requires repository settings to allow Actions to create PRs with GITHUB_TOKEN if that setting is disabled.

## 6) Required GitHub Settings

Make sure your repo has:

- Actions workflow permissions that allow GITHUB_TOKEN to create pull requests
- branch protection on your default branch so the automation branch still goes through review
- `contents: write` and `pull-requests: write` permissions in the workflow

GitHub documents that the "Allow GitHub Actions to create and approve pull requests" setting controls whether GITHUB_TOKEN can create PRs, and that branch protection rules can require reviews and checks before merge.

## 7) Tweaks for a Real Team Repo

### Better Destinations

Instead of a single runbook/ADR file, route by area:

- Android → `docs/runbooks/android-memory.md`
- backend → `docs/runbooks/backend-memory.md`
- architecture → `docs/adr/adr-memory.md`

### Better Dedupe

Use an LLM-assisted normalize step later, but keep this initial scorer deterministic.

### Better Safety

Before PR creation, add:

- secret scanning on generated files
- forbidden pattern checks
- max promoted items per run, e.g. 10

### Better Signal

Add weights for:

- touched path count
- linked failing test names
- linked issue IDs
- repeated user corrections

## 8) What to Do First

Use this exact setup, but start with:

- `threshold = 4.0`
- draft PRs only
- one reviewer
- max 5 promotions per run

That keeps noise down early.
