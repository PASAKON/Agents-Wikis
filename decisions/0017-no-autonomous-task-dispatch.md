# ADR 0017 — No autonomous task dispatch: our constraint is review, not execution

- **Date:** 2026-08-13
- **Status:** Accepted
- **Decider:** CEO, overruling the CTO's initial recommendation
- **Source:** Hermes Agent survey — see `reference/hermes-agent-vs-org.md`

## Context

Hermes Agent ships a Kanban **dispatcher**: a loop that ticks every 60s,
reclaims stale claims, promotes ready tasks, atomically claims one, and spawns
the assigned worker profile. No human in the loop. The CTO's first pass ranked
this as the single most valuable thing to copy, on the reasoning that our queue
only moves when a C-level is awake and pushing.

The CEO rejected it. Three arguments, all of which survive scrutiny.

### 1. Every candidate job on the CTO's own shortlist was a script, not an agent

The CTO proposed a "stage 1" of code-free, low-risk jobs. Re-examined:

| Proposed job | What it actually is |
|---|---|
| GC stale worktrees / sessions | shell script + cron |
| Run tests and report | CI |
| CVE scan | `osv-scanner` in CI |
| Wiki dead-link check | `lychee` in CI |
| Pull broker / metrics numbers | cron + script (n8n already runs on Contabo) |
| Retry a task that has a replay script | the replay script **is** the automation |

Six for six. The shortlist was selected for *low risk*, but low-risk work is
low-risk precisely because it needs no judgement, and work that needs no
judgement should not be paying for an LLM.

**The line, stated once:** if the steps can be written down in advance, it is a
script. If they cannot, because someone has to look at the situation first, it
is an agent.

### 2. Adding executors does not help a review-bound system

The org's visible backlog (12+ overdue LungNote to-dos, claudeflow CI red since
19 Jul, five never-invoked skills) is not blocked on someone writing code. It
is blocked on someone having the attention to **decide** about it.

A dispatcher that produced five finished jobs overnight would deliver five
unreviewed branches. The queue does not shorten; the inventory moves from
"not started" to "done but unverified", which is strictly worse because
branches rot against a moving `main`.

Capacity added upstream of the constraint becomes inventory, not throughput.

### 3. The 5 a.m. test

> **If you would not ask a hired engineer to do it at 5 a.m., do not ask an
> agent to.** — CEO, 2026-08-13

This is a better filter than the four-question rubric the CTO drafted, because
it resolves in two seconds without any technical reasoning.

### What the cost actually is (measured, since the premise was worth checking)

The CEO's stated worry was invisible API spend. Verified on Contabo
2026-08-13:

| Check | Result |
|---|---|
| `~/.claude.json` on the VPS | `oauthAccount = pass.gob1@gmail.com` — same Max subscription as the Mac |
| `ANTHROPIC_API_KEY` on the box | absent |
| `extra_usage.is_enabled` | **false** (`disabled_reason: out_of_credits`) |

So: moving work to Contabo adds **zero** marginal cost, and the financial
exposure is hard-capped at $0 because overage billing is off — when quota runs
out, Claude stops rather than billing on.

The real scarce resource is **subscription quota shared with the CEO's own
working hours** (at time of writing: 5h window 40% used, 7d window 32% used).
That is a genuine cost, and it is invisible, but it is bounded and it is not
dollars. Recording it here so the premise does not get re-litigated from
memory.

### No necessity case could be constructed

Three theoretical cases where autonomous dispatch is genuinely required, and
why none is ours:

| Case | Applies to us? |
|---|---|
| An externally imposed time window (an API only open at certain hours) | No — and even then, *collection* is a script; *interpretation* can wait for morning |
| Long-running work where wall-clock matters | No — that is one command issued before bed, not a daemon |
| Reacting to unpredictable events (3 a.m. prod alert) | No — that is on-call, not a work queue, and a prod incident wants a human anyway |

## Decision

1. **The org will not build autonomous task dispatch.** Not "later", not
   "stage 1". Tasks are pushed by a C-level who is present and accountable.
2. **Repeatable work goes to deterministic automation** — cron, shell, CI,
   n8n — never to a spawned agent. If a job can be written down, write it down.
3. **This ADR is the answer to the next person who proposes it.** Reopening
   requires a concrete job that passes the 5 a.m. test *and* cannot be
   expressed as a script.
4. **Cleanup automation is explicitly NOT covered by this ban.** Reclaiming
   stale claims, reaping dead worktrees, and auto-blocking a task after
   repeated failures are *janitorial* — they tidy work that already died, they
   do not originate work. `gc_stale_tasks.py` and `watchdog.py` stay, and may
   be extended with Hermes' `failure_limit` idea.

## Consequences

- The queue continues to require a present C-level. Accepted deliberately.
- Throughput improvements must target **review capacity**, not execution
  capacity. Anything proposed as "more agents" should be checked against this.
- `reference/hermes-agent-vs-org.md` §3 originally ranked the dispatcher as the
  #1 item to copy. That ranking is superseded by this ADR.
- The 5 a.m. test should be promoted into IRON-RULES as a standing filter.
