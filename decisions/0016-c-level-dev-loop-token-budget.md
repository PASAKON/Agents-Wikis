# ADR 0016 — The C-level ↔ DEV loop pays for silence, so stop buying it

- **Date:** 2026-08-12
- **Status:** Accepted
- **Decider:** CEO, on CTO analysis
- **Source incident:** `task-cda4f469` — a 10.5-hour Higgsfield generation wave

## Context

The wave delivered 18 clips at **$0 credit spend**, which was the hard
requirement. It cost three `browser_operator` instances (two killed) and an
enormous number of C-level turns for work that was almost entirely mechanical.

Measured from `state/logs/cto-bc7dfe09.log` and the CTO session:

| Signal | Count | Note |
|---|---|---|
| DEV → CTO messages | **65** | **52 carried no new information (80%)** |
| CTO → DEV messages | ~17 | one died on a 2-minute SIGTERM, message too long |
| `delegate_task` calls | 5 | each echoed the full ~4,000-word description |
| Operators used | 3 | 2 killed |

Two facts drive everything below.

**A heartbeat costs a full turn.** Each of those 52 contentless pings forced
the C-level to re-process its entire system prompt, every `CLAUDE.md`, the
memory index and the whole conversation, in order to emit one word of
acknowledgement. The input cost of "still rendering" is identical to the input
cost of real analysis.

**Two of the three operators died for an environmental cause.** A spawned
DEV's shell refuses standalone `sleep`, while `higgsfield-unlimited-gen`
instructed them to "use an active sleep-and-check loop". Neither could comply;
both fell back on a scheduled wake, which does not fire reliably for a
subprocess, and went silent. The rule made the failure mandatory.

## Decision

### 1. DEVs report on state change only; the C-level watches liveness itself

`roles/_dev_shared.md` Hard Rule 10. A DEV messages when work landed, work
failed, it is blocked, content was flagged, or two instructions conflict.
"Still rendering", "pacing 5 minutes", "will check on wake", "holding" are
failures, not reports.

`dev-spawn-protocol` §6b makes the replacement mandatory: arm a Monitor over
`state/tasks.db` at delegate time, emitting only on status change or on
`status='in_progress'` with a dead pid. The reference command is in that skill.
It caught a killed operator immediately during this wave and costs nothing per
quiet interval.

**Rejected alternative:** keeping a long-interval backstop ping (one every 30
minutes). The CEO chose full silence. The accepted residual risk is a process
that is alive but internally wedged, which the Monitor cannot see. Detect that
from the *absence* of expected state changes against the job's known per-item
duration — not by re-introducing heartbeats.

### 2. Repetition must produce a script, and the merge gate enforces it

`cto-merge-checklist` Gate 8: more than 3 repetitions of the same operation
with `Replay Script: none` in the report ⇒ refuse the merge.

The script was already required by `roles/browser_operator.md` and
`dev-spawn-protocol` §3c.8. Both were satisfied by writing `none`. A rule with
an opt-out phrase is not a rule. On this wave that omission cost roughly five
model turns per clip across twenty near-identical clips.

### 3. The MCP surface stops echoing task descriptions

`lib/org_tools_registry.py` gains `_slim_task()`, applied in `_h_get_task` and
`_h_delegate_task`. Measured on the incident's own task row: **20,473 bytes →
5,974, a 71% cut per call.** `get_task` takes `include_description=True` when
the text is genuinely wanted.

`tools/delegate.py` is deliberately untouched — it returns `db.get_task()` from
nine exit points, so the projection belongs at the boundary. Internal callers
keep the full row.

### 4. `reopen_task` prepends

Appending stacked mutually contradictory instructions into one document. This
task reached iteration 3 carrying the original rule plus two corrections to it,
read top-down by each respawned DEV, which acted on stale rules twice before
reaching the live one. It took a hand-written banner at the top of `TASK.md` to
work around.

### 5. Long content goes to the worktree, not the chat

`dev-spawn-protocol` §3c.9. `tools/send_to_dev.py` types character-by-character
into a TUI, can die mid-type, and is echoed back in full so the same text is
paid for twice. Proven on this wave: a `PROMPTS.md` in the worktree ended the
operator's repeated requests for prompt text permanently and survived the DEV
being killed and respawned.

### 6. Two new IRON-RULES

- **§43 — Suspect the environment before the agent.** Two identical failures in
  a row means the environment or the instructions. The cheapest test is to stop
  intervening and watch. Corollary: killing an agent does not cancel its
  in-flight work; a server-side job outlives its author.
- **§44 — A rule an agent can satisfy with an adjective is not a rule.** Write
  rules as values to produce, not qualities to have.

## Explicitly not changed

These look like savings and are not. All three earned their keep during this
wave:

- **The zero-digit Generate check before every click** — caught a live "130"
  price after a page reload silently reset the Unlimited toggle.
- **Reading the usage ledger after every generation** — the only thing that
  substantiates the $0 claim.
- **The DEV stopping to ask when it meets something new** — it stopped four
  times (concurrency toast, ledger discrepancy, "Prompt is required", stuck
  Unlimited toggle) and every stop saved money or time.

## Consequences

Expected: a comparable wave costs roughly a quarter of the C-level turns. The
wall-clock floor is unchanged, because it is set by Higgsfield's render time
(~26-30 min per clip on this project), not by us.

Success criterion for the next Higgsfield wave: **fewer than 20 DEV messages
for a 20-clip run** (this wave: 65), and a replay script committed under
`scripts/browser/`.

## Implementation

- `b598a66` — docs: Hard Rule 10, `dev-spawn-protocol` §6b and §3c.9, merge Gate 8
- `f54cf0f` — code: `_slim_task()`, `include_description`, `reopen_task` prepend
- Skill fixes made mid-wave that this ADR builds on: `4e723a9` (sleep blocked in
  DEV shells), `affa50a` (decoy contenteditable), `e4ee8a1` ("Prompt is
  required" with text present ⇒ use Recreate for repeats)

Tests: `scripts/test_tool_parity.py` (17 tools, all three surfaces) and
`scripts/test_org_tools_registry.py` both ALL PASS.
