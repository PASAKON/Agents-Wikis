# ADR 0018 — Skills get provenance, a lifecycle, and a curator that only touches what we own

- **Date:** 2026-08-13
- **Status:** Accepted
- **Decider:** CEO ("อยากให้มีมาก — คู่มือที่แก้ตัวเองได้ ไม่ต้องรอให้สั่ง")
- **Source:** Hermes Agent survey — `agent/curator.py`, `tools/skill_usage.py`

## Context

### What we already have (measured 2026-08-13)

The CTO's first report claimed we had no skill telemetry. That was wrong.
`scripts/hook-skill-log.py` is registered as a `PostToolUse` hook in
`.claude/settings.json` and has been appending to `state/skill-usage.log`
(TSV: timestamp, skill, session id) for some time. **351 records** at time of
writing.

The data is already actionable:

```
49  cto-merge-checklist        12  debug-mantra
44  session-close              12  artifact-design
42  ecc:save-session           11  higgsfield-unlimited-gen
38  dev-spawn-protocol          8  session-save
28  browser-operator            6  session-list
21  session-open                6  loop
13  gdrive-filing               5  scrutinize
```

Never invoked once, yet loaded into every session's context budget:
`mooniex-finance`, `session-restart`, `spawn-web-designer`, `terminal-open`,
`terminal-restart`.

### What is missing

Telemetry with nothing reading it is the same write-only graveyard `lib/recall.py`
was written to fix for tasks. Specifically absent:

| Missing | Consequence |
|---|---|
| **Provenance** — frontmatter carries `name/owner/origin/scope/description`, nothing says who authored it | No safe way to tell "ours to prune" from "someone else's" |
| **Lifecycle state** — no active / stale / archived, no pin | Dead skills never leave; useful-but-quiet skills are indistinguishable from dead ones |
| **A review pass** | Nobody ever looks at the log |
| **Self-improvement** | A skill that misfires stays wrong until a human happens to notice |

### The ownership hazard, which drives most of the design

Of the 42 skills in `~/.claude/skills`, **most are symlinks into repos we do
not own**: `external/wondelai-skills`, `external/9arm-skills`,
`external/marketingskills`, `.agents/skills`. Only the 19 under
`Agents/.claude/skills` are org-authored.

A curator that patched or archived a symlinked skill would be editing another
project's repo through a link. Hermes hit the same problem and solved it with a
hard invariant: the curator only touches skills whose provenance says
`created_by: agent`. We need the equivalent, and ours must additionally refuse
anything reached through a symlink.

## Decision

### 1. Frontmatter gains two fields

```yaml
created_by: human | agent      # who authored it; absent → treated as human
lifecycle: active | stale | archived | pinned
```

`created_by` is written once at authoring time and never rewritten by an agent.
Absent means `human`, so every existing skill is protected by default.

### 2. Usage telemetry gets a reader

A sidecar `state/skill-usage.json` derived from the existing TSV, carrying per
skill: `use_count`, `last_used_at`, `first_seen_at`, `lifecycle`, `pinned`.
The TSV stays as the append-only source of truth; the JSON is a rebuildable
projection. No new hook, no new write path in the hot loop.

### 3. Auto-transitions are proposals, not actions

Derived on read, never applied silently:

- unused for **30 days** → propose `stale`
- `stale` for a further **30 days** → propose `archived`
- `pinned` → exempt from every transition, forever

### 4. Curator invariants (non-negotiable, lifted from Hermes)

1. **Never deletes.** The most destructive action is archive, and archive is
   restorable.
2. **Only touches `created_by: agent`.** Human-authored skills are read-only to
   it.
3. **Refuses any path that resolves through a symlink.** This is ours, not
   Hermes': it is what keeps the curator out of `external/*`.
4. **Pinned skills are exempt** from transitions and from the review pass.
5. **Backs up before any mutation** — a snapshot that can be rolled back.

### 5. It runs when a human is present

Per [ADR 0017](0017-no-autonomous-task-dispatch.md) there is no daemon. The
curator runs as a slash command a C-level invokes, and `/session-close` may
surface "N skills are proposed for archive" as a one-line nudge. Applying a
proposal is always a human keystroke.

### 6. Agent-authored skills are allowed, and are the point

When a C-level finishes work that took real figuring-out and would recur, it
may write the recipe as a skill with `created_by: agent`. Those, and only
those, are what the curator later reviews and patches. This is the half of the
learning loop that does not exist today.

## Consequences

- Context budget shrinks: five never-used skills leave the always-loaded
  `name + description` set (see [ADR 0015](0015-skill-context-budget.md) for why
  that is the only part that costs).
- The org gains a written answer to "is this skill earning its place", which is
  currently pure guesswork.
- Combined with [ADR 0019](0019-memory-self-nudge-and-session-search.md), this
  closes the learning loop: do work → write skill → telemetry → review →
  patch or archive.
- Risk accepted: an agent writing skills can write bad ones. Mitigated by
  provenance (they are labelled), by review (the curator reads them), and by
  the fact that a bad skill is a bad *suggestion*, not a bad *action*.
