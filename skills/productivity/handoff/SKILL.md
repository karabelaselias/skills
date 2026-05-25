---
name: handoff
description: Compact the current conversation into a handoff document for another agent to pick up. When available, include Beads task state and Haft decision/governance context by reference.
argument-hint: "What will the next session be used for?"
---

# Handoff

Write a handoff document summarising the current conversation so a fresh
agent can continue the work. Save it to a path produced by
`mktemp -t handoff-XXXXXX.md` and read the empty file path before writing
to it.

If the user passed arguments, treat them as the next session focus and tailor the handoff accordingly. 
Redact any sensitive information, such as API keys, passwords, or personally identifiable information.

## Context discovery

Before writing the handoff, check for durable project context. Keep this
read-only unless the user explicitly asks you to update task state.

### Always useful

- Current working directory and repo root if available
- `git status --short` when inside a git repo
- Relevant changed files or diffs, summarized by path
- Skills the next session should load

### Beads, if present

Detect Beads by looking for `.beads/` in the current directory or an
ancestor. If present and `bd` is available, query minimal state:

```bash
bd status --json --no-activity
bd ready --json --limit 10
bd list --status in_progress,blocked --json --limit 20
```

If the handoff focus mentions a keyword or issue id, also query:

```bash
bd search "<focus>" --json --limit 10
bd memories "<focus>"
```

Use Beads for task tracking, active/blocked work, handoff notes, and
durable operational memories. Do not copy full issue bodies unless they
are necessary; reference bead IDs and include only the relevant status,
blocker, and next action.

### Haft, if present

Detect Haft by looking for `.haft/` in the current directory or an
ancestor. If present, prefer Haft MCP tools when available:

- `haft_query(action="status")` for active decisions/problems/staleness
- `haft_query(action="search", query="<focus>")` for focus-specific recall
- `haft_query(action="related", file="<path>")` for changed-file decisions

If MCP tools are unavailable, use shell fallbacks:

```bash
haft sync
haft check --json
find .haft -maxdepth 3 -type f | sort
```

Use Haft for decision records, problem frames, WorkCommissions,
governance debt, stale claims, and reasoning artifacts. Do not duplicate
DRRs/problem cards/specs; reference their paths or IDs and summarize only
what the next agent must know.

## Handoff format

Write concise Markdown with these sections:

```markdown
# Handoff: <short title>

## Next-session focus
<What the next agent should optimize for.>

## Current state
<What changed, what was learned, current repo/task status.>

## Durable context checked
- Beads: <not present | queried commands + key issue IDs>
- Haft: <not present | queried tools/commands + key artifact IDs/paths>

## Decisions and constraints
<Only unresolved or load-bearing decisions. Reference existing artifacts.>

## Files and artifacts
- `<path>` — <why it matters>

## Open questions / blockers
- <question or blocker>

## Recommended next steps
1. <action>
2. <action>

## Suggested skills for next agent
- `<skill>` — <why>
```

## Rules

- Do not duplicate content already captured in PRDs, plans, ADRs,
  issues, commits, diffs, Beads, or Haft. Reference by path, URL, bead
  ID, Haft artifact ID, or command output summary.
- If Beads/Haft queries fail, include the failure briefly and continue.
- Do not mutate Beads or Haft state during handoff unless explicitly
  requested.
- Prefer exact paths and IDs over prose memory.
- Keep the handoff small enough to be read at session start.
