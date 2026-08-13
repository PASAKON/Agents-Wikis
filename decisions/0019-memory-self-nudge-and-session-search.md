# ADR 0019 — Memory nudges itself, past sessions become searchable, LungNote may add but never delete

- **Date:** 2026-08-13
- **Status:** Accepted
- **Decider:** CEO
- **Source:** Hermes Agent survey — `hermes_state_search.py`, `memory` config block

## Context

### What we have

| Piece | State |
|---|---|
| Memory store | 125 files in `~/.claude/projects/-Users-gob-Projects-Agents/memory/`, one fact each, typed `user / feedback / project / reference`, indexed by `MEMORY.md` |
| Recall | `lib/recall.py` — keyword rank over `tasks.db`, injected by `scripts/hook-recall.py` on every prompt |
| Reflect | `lib/reflect.py` |
| Session transcripts | **137 `.jsonl` files, 2.4 GB, for this project alone** (132 projects total under `~/.claude/projects/`) |

Curation quality is genuinely good: each memory carries `**Why:**` and
`**How to apply:**`, links siblings with `[[name]]`, and converts relative
dates to absolute. Better than Hermes' flat `MEMORY.md`.

### The two real gaps

**1. Memory is written only when the model happens to think of it.**
There is no trigger. Hermes nudges itself every 10 turns
(`memory.nudge_interval: 10`, `flush_min_turns: 6`). We nudge never. The
failure is silent and unmeasurable: nothing records the facts that should have
been saved and were not.

**2. 2.4 GB of past conversation is unreadable.**
`recall()` searches `tasks.db` — task titles, reports, merge shas. It does not
touch the transcripts. So anything discussed but never turned into a task or a
memory file is gone the moment the context window rolls. In practice this
session alone re-derived facts that earlier sessions had already established.
Hermes indexes every message with SQLite FTS5 (plus a trigram index and a CJK
tokenizer) and searches it directly.

### The LungNote question the CEO raised

The CEO asked whether memory should also add and delete LungNote to-dos.
These are different objects and deserve different rules:

- **memory** = things that *are true* (`XM $15/lot`, `never use มึง/กู`).
  Long-lived, idempotent, cheap to be wrong about.
- **LungNote** = things that must be *done*. Carries deadlines and a done-state.
  It is the CEO's record of outstanding obligations.

A wrongly-written memory is noise a human corrects later. A wrongly-deleted
to-do is a commitment that vanishes with nobody aware it existed. The whole
design of `/session-close` is to re-surface pending items; silent deletion is
its exact inverse.

## Decision

### 1. A memory nudge, cadence-based and cheap

Every N turns (start at 10, tunable), a hook injects a single line asking
whether anything this session should be persisted. It costs one line of
context, it does not call a model, and it does not write anything itself. The
C-level decides.

Only fires when the session has produced at least a few turns of substance, to
avoid nagging short exchanges.

### 2. Session transcripts become searchable

Build a FTS5 index over `~/.claude/projects/**/*.jsonl`, exposed as a tool the
C-level can call (`search_sessions(query)`), returning date, session id, and a
snippet.

Constraints:
- **Read-only.** The index is a derived artifact; deleting it must lose nothing.
- **Incremental.** Re-index by file mtime, never a full 2.4 GB rescan.
- **This project first.** 132 projects exist; scope to the Agents project
  initially and widen only if it earns it.
- **Not injected automatically.** `recall` already spends context on every
  prompt (see [ADR 0016](0016-c-level-dev-loop-token-budget.md) on what a turn
  costs). Session search is *pull*, invoked when the C-level suspects a thing
  was discussed before.

### 3. LungNote automation: add yes, complete conditionally, delete never

| Action | Allowed | Condition |
|---|---|---|
| **add_todo** | ✅ yes | Standing CEO permission granted 2026-08-03. Worst case is a junk to-do, trivially removed. |
| **complete_todo** | ⚠️ conditional | Only with machine-checkable evidence — a merge sha, a green test, a verified prod query. Must `append_note` the evidence. Never on model judgement alone. |
| **delete_todo** | ❌ never | A false delete is a silently dropped commitment. Only the CEO deletes. |

This is a hard rule, not a default.

## Consequences

- Fewer facts lost between sessions; fewer questions re-asked that were
  answered weeks ago.
- Context cost of the nudge is one line per N turns. Session search costs
  nothing until called.
- The index is a new derived artifact to keep from going stale; incremental
  re-index by mtime is the mitigation. Related failure worth remembering:
  never dedup or compare against a store a live process is still writing
  (2026-08-06, ~20 GB miss).
- Together with [ADR 0018](0018-skill-lifecycle-and-curator.md) this closes the
  learning loop. Neither ADR needs a daemon; both fire while a human is present,
  consistent with [ADR 0017](0017-no-autonomous-task-dispatch.md).
