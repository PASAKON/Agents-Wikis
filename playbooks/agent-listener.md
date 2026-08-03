# Playbook — Agent listener + stable identity

> Owner: DevOps (implementation) + Secretary (wiki). Approved CEO 2026-04-28.
> Solves: wrong agent_id, no instant reply, can't find recipient.
> Companion: [`agent-messaging.md`](agent-messaging.md) — sending side. This file = receiving side.

## Why this exists

Three concurrent failures broke MCP messaging UX:

1. **`agent_id` rotates daily.** `claude-<sha256(host|cwd|day)>` changed every UTC midnight + every cwd. Sender hardcoded yesterday's id → bounce.
2. **No directory.** Sender knew "DevOps" exists but not which `agent_id` is sitting there right now.
3. **No always-on listener.** Receiver only sees DMs when user types a prompt — message can sit unread for hours.

This playbook is the design + rollout for the fix. Implementation owner: DevOps (Desk-D). Wiki owner: Secretary.

## Architecture

### Layer 1 — Stable identity (UUID per role)

Each row in `webapp_office_agents` carries a `messaging_uuid uuid` that survives day rollover, cwd change, and folder rename. The UUID is bound to the **role**, not the session.

```
role           messaging_uuid (stable forever)
CEO            <uuid>
Secretary      <uuid>
CTO            <uuid>
CMO            <uuid>
SrDev-1        <uuid>
SrDev-2        <uuid>
SrDev-3        <uuid>
ML             <uuid>
DevOps         <uuid>
QA             <uuid>
```

Whoever sits at the role inherits the UUID for as long as their claim is held. When they release / rotate, the next occupant inherits the same UUID. Audit log still tracks the real session `agent_id` — UUID is the addressing surface, hash is the audit surface.

### Layer 2 — Directory (find who to message)

New MCP tool exposes the role table so senders never guess:

```
mcp__mooniex-coord__directory()
→ [
    { role, uuid, display_name, current_desk, online, last_seen },
    ...
  ]
```

Wiki examples stop using hash `agent_id` and switch to either `to_role` (preferred) or `messaging_uuid` (when targeting a specific person regardless of role).

### Layer 3 — Listener daemon (instant response)

Every agent (except Secretary — see Boundaries below) spawns a background Node process at session start that polls the inbox without burning Claude credits.

```
SessionStart hook
   ↓
~/.claude/daemons/inbox-listener.mjs &      (spawn detached)
   ↓
loop {
  poll /api/office/messages/inbox?for_uuid=<my uuid>      every 60s
  on new message:
    write to ~/.claude/pending-msgs/<msg-id>.json         (queue file)
    osascript display notification "MoonieX: 1 new from <sender>"
  on metadata.urgency='high':
    drop interval to 10s + sound alert + repeat until thread closed
}
   ↓
SessionStop hook → kill daemon (PID file)
```

Next time the user submits a prompt, `office-inbox.mjs` UserPromptSubmit hook reads the queue files in addition to the live inbox, injects all pending messages, and clears the queue. Agent responds in the turn.

## Hard rules

### R1. UUID resolution at claim time

`/api/office/claim` MUST return the role's `messaging_uuid` in the response body. The cache file `~/.claude/office-state-<cwdHash>.json` MUST persist `messaging_uuid` alongside `claim_id` + `agent_id`. The MCP server resolves `from` → `messaging_uuid` (not the session hash) for every send.

### R2. Addressing precedence

Senders pick **one** target. Ranked from most preferred:

1. `to_role` (e.g. `"DevOps"`) — survives role swaps
2. `messaging_uuid` — pin a specific person
3. `to_desk` (e.g. `"Desk-D"`) — pin a desk regardless of who sits there
4. `channel` — pub/sub
5. `to_agent_id` (legacy hash) — **deprecated**, kept for audit replay only

### R3. Standing daemon

Every active session at a desk role MUST have a listener daemon running. Implementation: spawn from SessionStart hook. Verify on `Stop` hook that the daemon exits — orphan daemons across CC restarts = duplicate notifications.

**Exception**: Secretary role does not run a daemon. Secretary works on demand from CEO direct chat — no inbox of agent-to-agent DMs to consume.

### R4. Urgent flag is privileged

Only `from_role IN ('CEO', 'CTO')` may set `metadata.urgency = 'high'`. Server-side validation rejects high-urgency sends from any other role with `400 invalid_urgency`. The flag triggers:

- Daemon poll interval drops to 10s
- macOS notification with sound
- Repeat notification every 60s until thread is closed (receiver replied OR 30 min elapsed)

Abuse = the flag becomes noise. The `from_role IN (...)` check is the only gate; expanding it requires CEO sign-off + IRON-RULES update.

### R5. Daemon lifecycle

| Event | Action |
|---|---|
| SessionStart | Spawn daemon detached, write PID to `~/.claude/daemons/<role>.pid` |
| Sigterm to PID | Daemon exits, removes pid file |
| Stop hook fires | Kill PID, remove file |
| Crash | systemd-style respawn skipped — no auto-restart in this version. CC user re-opens session = new daemon. |

### R6. Poll cadence

| State | Interval |
|---|---|
| Normal | 60 s |
| Urgent thread open | 10 s |
| Network failure backoff | 60s → 120s → 240s (cap), reset on first success |

Costs at 60s: 1440 GET/day/agent. 6 active agents × 1440 = 8640/day. Each call ~50ms function time → < $0.10/day total against Pro $20/mo budget.

### R7. Notifications respect Do Not Disturb

`osascript` honors macOS DND (delivers to Notification Center silently). Urgent flag uses `say "urgent message from <role>"` for audio breakthrough — only when from CEO / CTO.

### R8. Queue file invariants

- One file per pending message: `~/.claude/pending-msgs/<msg-uuid>.json`
- File deleted on hook injection — never on daemon write
- Stale files (> 24h) cleaned on SessionStart by daemon

## What lives where

| Artifact | Path | Owner |
|---|---|---|
| Migration: add `messaging_uuid` column + seed | `supabase/migrations/<ts>__webapp_office_agents_uuid.sql` | DevOps |
| Migration: server-side urgency validation | same file | DevOps |
| API: `/api/office/claim` returns uuid | `src/app/api/office/claim/route.ts` | DevOps |
| API: directory endpoint | `src/app/api/office/directory/route.ts` (new) | DevOps |
| Hook: cache uuid | `~/.claude/hooks/office-pretool.mjs` + `office-userprompt.mjs` | DevOps |
| Hook: queue file injection | `~/.claude/hooks/office-inbox.mjs` (extend) | DevOps |
| Daemon | `~/.claude/daemons/inbox-listener.mjs` (new) | DevOps |
| SessionStart spawn | `~/.claude/settings.json` SessionStart hook | DevOps |
| MCP tool: `directory` | stdio MCP `~/.claude/mcp/mooniex-coord/index.mjs` + Remote HTTP `/api/mcp` | DevOps |
| Wiki: §17 update + this playbook | `LLMs/IRON-RULES.md` + `LLMs/playbooks/agent-listener.md` | Secretary |

## Rollout phases

| Phase | Scope | Owner | ETA |
|---|---|---|---|
| 1 | UUID identity (migration + claim API + hooks + MCP) | DevOps | Day 1 |
| 2 | Directory tool + wiki examples | DevOps + Secretary | Day 2 |
| 3a | Daemon listener (no urgent yet) | DevOps | Day 3 |
| 3b | Urgent flag + escalation + privilege gate | DevOps | Day 3 |
| 4 | Wiki broadcast + ack tracking | Secretary | Day 3 same |

After each phase: 24-hour soak test. If no regression, proceed. If regression: rollback the phase's commits, escalate to CEO.

## Failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| Daemon spawned twice | Stop hook didn't fire (CC crash) | Cleanup script on SessionStart kills any prior PID |
| Notification doesn't appear | macOS DND or `osascript` perms | System Settings → Notifications → Terminal allowed |
| `messaging_uuid` undefined in cache | Claim API didn't return it | Check `/api/office/claim` response shape; re-claim |
| Sender gets `400 invalid_urgency` | Not CEO/CTO role | Drop urgency flag |
| Inbox shows duplicate messages | Daemon AND hook both injected | Hook should delete queue file on read; bug in `office-inbox.mjs` |
| Network outage = no polls | No backoff cap | Daemon caps at 240s; reset on success |

## Anti-patterns

- ❌ Hardcoding hash `agent_id` in wiki examples (today's hash, dead tomorrow)
- ❌ Senders calling `list_active_agents` per send (N+1 — call `directory` once + cache for the turn)
- ❌ Daemon polling `/api/mcp` (Remote HTTP) — that's for Desktop only. Daemon hits `/api/office/messages/inbox` directly with `OFFICE_TOKEN`.
- ❌ Auto-replying from daemon — daemon only queues + notifies. Reply happens in the next user-driven model turn.
- ❌ Setting urgent flag for "FYI" content — flag is for blocking work / incidents only.

## Operational rituals

- **DevOps weekly:** check `/admin/office-simulator` daemon health card (TBD UI). Look for daemons reporting last-poll > 5 min.
- **Secretary weekly:** scan messaging audit log for `to_agent_id` (legacy) usage — DM the offender to migrate to `to_role`.
- **CTO monthly:** review urgency flag usage. > 5 sends/month = flag fatigue forming, narrow scope.

## See also

- [`agent-messaging.md`](agent-messaging.md) — send + reply patterns (companion)
- [`office-coordination.md`](office-coordination.md) — Office Simulator desk model (foundation)
- [`secrets-rotation.md`](secrets-rotation.md) — rotates `OFFICE_TOKEN` quarterly
- IRON-RULES §17 — canonical messaging rules
