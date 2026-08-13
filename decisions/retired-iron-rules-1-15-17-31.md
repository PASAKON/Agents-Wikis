# Retired IRON-RULES sections (§1, §15, §17, §31)

- **Retired:** 2026-08-14
- **Decider:** CEO — "ตัดได้เลย มันควรอยู่ใน Skills มากกว่า แต่ฉันไม่ได้ใช้นานแล้ว"
- **Removed from:** `org:IRON-RULES.md` (1,122 → 733 lines, −389 lines / ~35%)

## Why each was retired

| § | Rule | Evidence it was dead |
|---|---|---|
| **1** | One repo = one folder (desk model) | Superseded by [ADR 0007](decisions/0007-deprecate-desk-pattern.md). The still-live half (one folder per repo, parallel via `git worktree`) is already stated in `/Users/gob/Projects/CLAUDE.md`, which loads on every session in that tree — so nothing is lost by removing it here. |
| **15** | Agent role memory + Office Simulator coordination | No `office_simulator` code remains in the runtime. The webapp page `admin/office-simulator` was last touched 2026-04-27. Worse than dead: §15 instructed agents never to read another agent's memory, but every C-level has shared one memory store (keyed by working directory, not role) for months — the rule contradicted the running system. |
| **17** | Agent-to-agent messaging (Office mailbox) | No `send_message` / `mailbox` / `inbox` tool survives in `lib/org_tools_registry.py`. Cross-agent chat runs through `tools/send_to_dev.py` and `tools/send_to_cxo.py` instead. |
| **31** | One CTO per repo (parallel-CTO sharding) | Not practised. Sessions shard by time, not by repo — a single session routinely spans the org repo, both wikis, and GitHub. CEO confirmed it has been unused for a long time. |

## What replaced them

Nothing needs to replace §15/§17 — the systems they governed no longer exist.
§1's live content lives in `/Users/gob/Projects/CLAUDE.md` and
[ADR 0007](decisions/0007-deprecate-desk-pattern.md).

Per the CEO's framing, rules of this kind — long procedural descriptions of one
subsystem — belong in a **skill that loads on demand**, not in a file every
session pays for. IRON-RULES should hold only what every agent must know before
it acts.

## Dangling references left behind

These still point at the retired sections and need cleanup or deprecation:

- `CHEATSHEET.md` — links §17 and §1
- `playbooks/office-claim-api.md` — entirely about the retired desk/claim system; its own header already said "delete once §15 is rewritten"
- `playbooks/agent-messaging.md` — entirely about the retired mailbox
- `playbooks/worktree-and-concurrent-sessions.md` — links "§1 desk model"
- `playbooks/projects-tasks.md`, `playbooks/secrets-rotation.md`, `playbooks/claudeflow-mesh.md`, `playbooks/cron-rules.md`, `playbooks/agent-listener.md` — passing citations

---

# Verbatim text, as removed


## Section 1 — One repo = one folder (parallel via branches + GH issues)

**Deprecated as of 2026-05-28: the multi-desk / "Desk-A..F" folder model.**
See [`decisions/0007-deprecate-desk-pattern.md`](https://github.com/PASAKON/Agents-Wikis/blob/main/decisions/0007-deprecate-desk-pattern.md) (`org:decisions/0007-deprecate-desk-pattern.md`)
for the rationale.

`mooniex-webapp` is **one repo** (`PASAKON/mooniex-webapp` on GitHub) checked
out to **one canonical folder**: `/Users/gob/Projects/mooniex-webapp/`. Every
agent works from that single folder.

### Hard rules

1. **One canonical clone per repo.** Path: `/Users/gob/Projects/<repo-name>/`.
   No spaces, no parens, no `(Desk-X)` suffix.
2. **Parallel work uses branches, not folders.** Each task gets its own
   branch (`claude/<feature>`, `codex/<feature>`, `agent/<role>-<task-id>`).
   Branches diverge, get reviewed via PR, merged via `merge_task` / CTO.
3. **Conflict prevention via GitHub issue assignment.** Before starting work
   on a hot file (anything in §1.4 below), open or claim a GH issue tagged
   with that file's path. One assignee per hot file at a time. Other agents
   wait or pick a different scope.
4. **Hot files (need issue claim before edit):**
   - `src/middleware.ts`, `src/app/layout.tsx`, `src/app/page.tsx`
   - `src/lib/supabase/server.ts`, `src/lib/supabase/client.ts`
   - `next.config.mjs`, `tailwind.config.ts`, `tsconfig.json`
   - `package.json`, `pnpm-lock.yaml`
   - Any cross-cutting `<TopBar>` / `<Sidebar>` / `<ThemeProvider>`
5. **Need parallel checkouts on one machine?** Use `git worktree`, not a
   second clone:
   ```bash
   cd /Users/gob/Projects/mooniex-webapp
   git worktree add ../mooniex-webapp-wt-<feature> <branch>
   ```
   Worktrees share the same `.git` dir — no duplicate node_modules, no
   stale remote drift. Delete with `git worktree remove`.
6. **Pull before edit, every time** — `git fetch origin && git pull --ff-only`.
7. **Push immediately after commit** — never leave local commits sitting.
   The remote is the source of truth other agents pull.
8. **Never run two Claude sessions on the same folder.** The PreToolUse hook
   hashes `(hostname + cwd + day)` to derive `agent_id`; two sessions with
   the same triple collide. For sub-agents, use `Agent` tool with
   `isolation: "worktree"`. See
   [playbooks/worktree-and-concurrent-sessions.md](https://github.com/PASAKON/Agents-Wikis/blob/main/playbooks/worktree-and-concurrent-sessions.md) (`org:playbooks/worktree-and-concurrent-sessions.md`).

### Folder layout (canonical)

| Repo | Folder |
|------|--------|
| `PASAKON/mooniex-webapp` | `/Users/gob/Projects/mooniex-webapp/` |
| `PASAKON/mooniex-claudeflow` | `/Users/gob/Projects/mooniex-claudeflow/` |
| `PASAKON/mooniex-claudesign` | `/Users/gob/Projects/mooniex-claudesign/` |
| (other repos) | `/Users/gob/Projects/<repo-name>/` |

Per-repo `CLAUDE.md` (checked into the repo) carries repo-specific rules.
Project-wide context lives in `LLMs/projects/<repo-name>.md`.

### Role assignment (replaces the desk-DB pattern)

Roles are assigned **per task**, not per folder. The Agents virtual-org
(`config/projects.yaml`) declares allowed worker roles per project; the CTO
spawns the right role for the work. The role is part of the task record,
not the working tree. The old `webapp_office_agents` table / `/api/office/claim`
endpoint is retired — see [`MIGRATIONS.md`](https://github.com/PASAKON/MoonieX-Wikis/blob/main/MIGRATIONS.md) (`mooniex:MIGRATIONS.md`) §desk-pattern.

### Why this model (replacing the desk pattern)

- A single canonical folder eliminates 6+ duplicate node_modules trees
  (~5 GB reclaimed in the 2026-05-28 cleanup) and stale remote drift
  (six of seven desks still pointed at the old `golfmaichai1` user).
- GitHub issues + branch-per-task already serialise edits the way desks
  were meant to. PRs review what the desks reviewed implicitly.
- `git worktree` covers the rare case where two checkouts ARE needed
  on one machine, without paying the disk + sync tax of a second clone.
- Audit trail unchanged: `git log --author` + commit messages on the
  shared remote.

---

## Section 15 — Agent role memory + Office Simulator coordination

> ⛔ **RETIRED 2026-05-28 — everything below is history, not instruction.**
> [ADR 0007](https://github.com/PASAKON/Agents-Wikis/blob/main/decisions/0007-deprecate-desk-pattern.md)
> removed the desk model. `webapp_office_agents` is not read,
> `/api/office/claim` is not called, `/admin/office-simulator` assigns
> nothing, and no folder named `mooniex-webapp (Desk-X)` exists on disk.
> The claim / heartbeat / release lifecycle described below does not run.
>
> **What holds now:** a session's role is the role it was spawned as
> (`roles/<role>.md` + `config.agents()`); one repo lives in one folder;
> parallel work uses branches and `git worktree`; a task is claimed through
> its GitHub issue and its `tasks` row. See §1.
>
> **The one rule here that survives, and still binds:** an agent reads only
> its own auto-memory — never another agent's
> `~/.claude/projects/<other-cwd>/memory/`. The reasoning at the end of this
> section is unchanged: two agents adopting one identity collide and corrupt
> the audit trail.
>
> Kept rather than deleted because §17 and §23 still use this section's
> vocabulary, and because a rule people may have memorised should be
> visibly cancelled rather than quietly disappear.

Goal: every Claude session has a **named role** (CTO, Senior Developer, ML Engineer, DevOps Engineer, QA Engineer) so its work doesn't collide with another agent's. The Office Simulator (`/admin/office-simulator`) is the live source of truth for who's at which desk; the agent's auto-memory holds the role brief the agent quotes when asked.

### Hard rules (agent role)

1. **Roles are admin-assigned only.** An agent NEVER changes its own role. Self-promotion or self-rebranding = rule violation.
2. **An agent reads ONLY its own auto-memory.** Never open another agent's `~/.claude/projects/<other-cwd>/memory/`. Roles are per-agent and isolated by design.
3. **When asked "your role / position?", quote the assigned `role_brief`** verbatim, then describe responsibilities. Example:
   > "ผมรับหน้าที่ในตำแหน่ง CTO — รับผิดชอบ technical roadmap ทั้งองค์กร Mooniex (architecture, infra, security, performance, code quality). ขอบเขตผมคือ: …"
4. **If no role assigned**, answer truthfully: "ยังไม่ได้รับมอบหมายตำแหน่ง — admin ยังไม่ assign ใน /admin/office-simulator."
5. **Role changes** happen ONLY when:
   - The user explicitly says "change your role to X" — write to memory + acknowledge
   - The admin updates `webapp_office_agents` via `/admin/office-simulator`. The next session's claim response includes the new role; agent updates its memory.
6. **Memory file convention**: each Claude session writes its role to `~/.claude/projects/<sanitized-cwd>/memory/reference_office_role.md`. Format:
   ```yaml
   ---
   name: Office role — <agent_id>
   type: reference
   ---
   ## Role
   <role>           # e.g. "CTO"

   ## Brief (quote when asked)
   <role_brief>

   ## Responsibilities
   - …
   - …

   ## Scope tags
   <comma list>

   ## Assigned by
   <email> on <timestamp>
   ```

### Quickstart — first session checklist

Before the first `Write` / `Edit` / `Bash` that mutates anything,
every Claude session runs:

```bash
# 1. confirm desk folder
basename "$PWD"               # expect: mooniex-webapp (Desk-X)

# 2. read assigned role
cat ~/.claude/projects/*/memory/reference_office_role.md | head -20

# 3. pull-before-edit
git fetch origin && git pull --ff-only

# 4. verify office API config
cat ~/.claude/office.json
```

If cwd is a `mooniex-webapp (Desk-*)` folder, the PreToolUse hook
auto-claims on first tool call — do nothing else, start working.
If cwd is elsewhere (CTO in `LLMs/`, ad-hoc scripts), the agent
must call `POST /api/office/claim` itself. Full commands +
role→desk catalog in
[`playbooks/office-claim-api.md`](https://github.com/PASAKON/Agents-Wikis/blob/main/playbooks/office-claim-api.md) (`org:playbooks/office-claim-api.md`).

### Role → desk catalog (canonical, set by CEO 2026-04-27)

| Role | Desk | Folder |
|------|------|--------|
| Senior Developer (slot 1) | `Desk-A` | `mooniex-webapp (Desk-A)/` |
| Senior Developer (slot 2) | `Desk-B` | `mooniex-webapp (Desk-B)/` |
| ML Engineer | `Desk-C` | `mooniex-webapp (Desk-C)/` |
| DevOps Engineer | `Desk-D` | `mooniex-webapp (Desk-D)/` |
| QA Engineer | `Desk-E` | `mooniex-webapp (Desk-E)/` |
| Senior Developer (slot 3) | `Desk-F` | `mooniex-webapp (Desk-F)/` |
| CTO | `Desk-CTO` | (cross-desk, no folder; claim via API) |

Match the role memory file to the desk above. Never sit at a desk
that doesn't match your role (§15 #1 rule violation).
**Desk-A / B / C are Senior Developer slots ONLY** — agents with
other roles must claim a different desk or escalate to CEO.

### Hard rules (desk coordination — per-turn lifecycle, 2026-04-27 update)

7. **Claim per turn — every UserPromptSubmit.** The UserPromptSubmit hook ensures a claim exists at the start of every assistant turn — heartbeats if cached, claims fresh if stale or absent. This applies to **every** Claude session that has a role assigned (via folder match OR role memory file), regardless of whether the turn does any edits. See [`playbooks/office-claim-api.md`](https://github.com/PASAKON/Agents-Wikis/blob/main/playbooks/office-claim-api.md) (`org:playbooks/office-claim-api.md`) for the full request shapes.
8. **Heartbeat is per-turn.** Hook beats once per UserPromptSubmit (throttled to 60s minimum interval to avoid spam during fast back-and-forth). PreToolUse beats opportunistically as a safety net for sessions where UserPromptSubmit didn't fire (e.g. /resume mid-edit). Manual / autonomous sessions hit `POST /api/office/heartbeat` themselves.
9. **Release per turn — every Stop.** The Stop hook releases the claim at the end of every assistant turn with `reason=turn_end`. Desk goes idle in `/admin/office-simulator` until the next UserPromptSubmit fires the next claim. Stop also covers `/clear`, `/resume`, compact, and session-end as backstops.
10. **One agent per desk** (already §1) is enforced at the DB layer via a unique partial index — concurrent claims fail deterministically. A 409 means switch desks, never bypass.
11. **Roles ≠ desks.** Same desk can be claimed by different agents on different days; the role comes back from `webapp_office_agents` joined into the claim response. Quote `agent_role` / `role_brief` from the response, not the desk letter.
12. **Cross-desk roles (CTO, CMO) — memory-based desk resolution.** When cwd doesn't match any `mooniex-webapp (Desk-*)` folder, the hook reads the role memory file at `~/.claude/projects/<sanitized-cwd>/memory/reference_office_role.md` and parses the `## Desk` line. CTO sitting in `/Users/gob/projects/LLMs/` still claims `Desk-CTO` because the memory file says so. agent_id formula switches to `sha256(host|"role:Desk-X"|UTC-day)` so cwd-hopping within the same day yields a stable id.

### Per-turn lifecycle diagram

```
turn 1: UserPromptSubmit → claim    →  ...edits / reads...  →  Stop → release (turn_end)
        (desk = working)                                          (desk = idle)
turn 2: UserPromptSubmit → claim    →  ...idle answer...    →  Stop → release (turn_end)
turn 3: UserPromptSubmit → claim    →  ...long edit batch...→  Stop → release (turn_end)
                                          ↑
                                  PreToolUse heartbeats opportunistically
                                  if turn lasts > 60s without prompt
```

Implications:
- `/admin/office-simulator` shows real-time idle / working state per agent
- Desk holders rotate: agent A claims Desk-A turn 1, releases, agent B can claim Desk-A turn 2 — both legit
- Stale-sweep cron (20 min) only fires for crashed sessions where Stop hook didn't run

### What lives where

| Artifact | Path |
|----------|------|
| Schema | `supabase/migrations/202604290100__webapp_office_simulator.sql` |
| API | `src/app/api/office/{claim,heartbeat,release,state}/route.ts` |
| Cron | `src/app/api/cron/office-sweep/route.ts` (hourly via `vercel.json`) |
| Admin UI | `/admin/office-simulator` |
| Admin role API | `src/app/api/admin/office/agent/[agent_id]/route.ts` (GET/PATCH, admin-gated, audit-logged) |
| Hooks | `~/.claude/hooks/office-{pretool,userprompt,stop}.mjs` |
| Hook config | `~/.claude/office.json` (api_base + token) |

### Activation env

| Where | Key | Value |
|-------|-----|-------|
| Vercel (preview + prod) | `OFFICE_TOKEN` | `printf "<random>" \| vercel env add OFFICE_TOKEN production` |
| Local `~/.claude/office.json` | `{ api_base, token }` | matches the Vercel value |

The same `OFFICE_TOKEN` gates all write endpoints. Reads (`GET /api/office/state`) are public so the admin UI works without an admin session for inspection.

### Why "never read another agent's memory"?

- Auto-memory lives at `~/.claude/projects/<cwd-hash>/memory/` — already isolated per CWD by Claude Code. Reading it directly across CWDs requires deliberate effort.
- Roles map to **desk × agent** identity. If agent A reads agent B's memory and adopts B's role, two agents claim the same scope → work collides + audit trail is wrong.
- The Office Simulator owns role assignment in DB. Memory is a cache. Source of truth = DB.

---

## Section 17 — Agent-to-agent messaging (Office mailbox)

ClaudeCode sessions are isolated processes — no native cross-session IPC. Agents coordinate via a shared **DB mailbox** in `webapp_office_messages` (migration `202604290500__webapp_office_messages.sql`). Each agent's `UserPromptSubmit` hook polls the inbox at the start of every turn and injects unread messages into Claude's context. Agents reply via the `mooniex-coord` MCP server tools (or a direct `curl` POST as fallback).

### Architecture

```
Session A                         Supabase                     Session B
  Claude → tool call                                              
  send_message(to:B, body)  →  webapp_office_messages  →  UserPromptSubmit hook
                                INSERT row                      reads unread
                                                               injects via additionalContext
                                                             → Claude B sees in next turn
```

No central server. DB is shared state. MCP server runs **stdio per session** — each ClaudeCode process spawns its own copy.

### Hard rules

1. **Identity = `messaging_uuid`, not session hash.** Each `webapp_office_agents` row carries a stable `messaging_uuid uuid` bound to the role. Resolved at `/api/office/claim` time and cached in `~/.claude/office-state-<cwdHash>.json`. The session hash `agent_id` is for audit only; senders address by **role** (preferred) or `messaging_uuid`. **Never spoof another agent's identity** — the `webapp_audit_log` records every send.
2. **Targeting is exactly-one-of** `to_role` | `messaging_uuid` | `to_desk` | `channel` | `to_agent_id` (legacy/deprecated). Enforced by DB CHECK. Addressing precedence (most preferred first): `to_role` → `messaging_uuid` → `to_desk` → `channel`. Use `to_agent_id` only for replaying historical sends.
3. **Body is 1–4000 chars.** Long context → store in a wiki page or commit and link.
4. **Inbox auto-injection** is silent and one-shot — once the hook ack's a message, it's marked read. Re-read via `read_thread` MCP tool.
5. **Threading**: pass `replied_to: <message_id>` to keep a conversation. Server resolves `thread_root` from the parent.
6. **Channels** (`#deploys`, `#incidents`, etc) are **NOT auto-pulled** by the inbox hook — pub/sub. Agents subscribe via tool calls.
7. **Broadcasts** (`to_role`, `to_desk`) reach every qualifying agent. Use sparingly.
8. **Standing listener daemon** — every active session (except Secretary) runs `~/.claude/daemons/inbox-listener.mjs` spawned at SessionStart. Polls 60 s (10 s on urgent), pushes macOS notification, queues messages for next-prompt injection. See [`playbooks/agent-listener.md`](https://github.com/PASAKON/Agents-Wikis/blob/main/playbooks/agent-listener.md) (`org:playbooks/agent-listener.md`). Secretary exempt — works on direct CEO chat only.
9. **Urgent flag is privileged** — `metadata.urgency = 'high'` may only be set by `from_role IN ('CEO', 'CTO')`. Server validates and returns `400 invalid_urgency` for any other role. Triggers 10 s daemon polling + audio breakthrough until thread closed (or 30 min timeout).
10. **Discovery via `mcp__mooniex-coord__directory()`** — when sender doesn't know the target's `messaging_uuid`, call `directory()` first, then send. Don't guess; don't hardcode hash `agent_id` in wiki examples.
11. **No secrets in `body` or `metadata`.** Same redaction rule as §6 logging.
12. **Acks are idempotent.** `read_by_agent_id` records who ack'd.
13. **Auth**: `Authorization: Bearer ${OFFICE_TOKEN}` (same env as desk claim — no new secret).

### What lives where

| Artifact | Path |
|----------|------|
| Schema | `supabase/migrations/202604290500__webapp_office_messages.sql` |
| Lib | `src/lib/office/messages.ts` |
| API | `src/app/api/office/messages/{send,inbox,[id]/ack,thread/[rootId]}/route.ts` |
| Hook | `~/.claude/hooks/office-inbox.mjs` (UserPromptSubmit) |
| MCP server | `~/.claude/mcp/mooniex-coord/index.mjs` (stdio, per-session, zero deps) |
| Hook + MCP config | `~/.claude/settings.json` + `~/.claude/.mcp.json` |
| Admin UI | `/admin/office-simulator` (speech bubbles + thread drawer — UI follow-up) |

### MCP tools

| Tool | Purpose |
|------|---------|
| `mcp__mooniex-coord__send_message` | DM / role / desk / channel send |
| `mcp__mooniex-coord__read_inbox` | pull unread + optional auto_ack |
| `mcp__mooniex-coord__read_thread` | full thread by root_id |
| `mcp__mooniex-coord__list_active_agents` | who's at which desk now |

### Mandatory usage protocol (CEO directive 2026-04-27)

**Every agent — folder DEV, role-bound (CTO/CMO), Desktop user — follows this protocol.** Violations break cross-surface coordination.

#### 1. Agent identity is mandatory

- **Resolve `from_agent_id` once at session start, then reuse.**
  - Folder DEV (Desk-A..F): `claude-<sha256(host|cwd|day)>` — auto-set by `office-pretool.mjs`
  - Role-bound (CTO): `claude-cto-cross-desk` — fixed identifier
  - Claude Desktop user: `claude-desktop-<short_label>` (e.g. `claude-desktop-ceo`) — caller must pass explicitly
  - Curl / scripts / external tools: pick a stable id (`script-<purpose>`) and document it in your project file
- **Anonymous sends are forbidden.** `webapp_audit_log` records every send — an unattributable id breaks incident forensics.
- **Never reuse another agent's id.** If you don't know yours, run `mcp__mooniex-coord__list_active_agents` and read your row, or read `~/.claude/office-state-<cwdHash>.json`.

#### 2. Inbox-first at session start

**Before doing any other work, every agent runs `read_inbox` (with `auto_ack: true`) for its `from_agent_id`.**

For folder DEVs the `office-inbox.mjs` UserPromptSubmit hook does this automatically — no manual step. For Desktop / external callers without the hook: explicit `read_inbox` call **first**, before tool calls or code edits.

Hook output appears as `Office Simulator inbox — N unread message(s)` in the model's context. Treat it as the first instruction of the turn — outranks user prompt for handling order, but does not replace it.

#### 3. Reply or complete — no orphan threads

For each unread message:

| Sender intent | Receiver action |
|---------------|-----------------|
| Question / ack request | **Reply once** with `replied_to: <msg_id>` — confirm receipt + answer |
| Task / instruction | **Execute the task** to completion, then **reply once** with the outcome (success / blocked / handoff) |
| FYI / broadcast | **Read** — reply only if you have a follow-up |
| Cross-desk handoff | **Read** + **claim the new scope** + **reply** with start time / branch |

**A thread is closed when the requester reads the final reply.** If you ack but don't finish the task, you owe a second message when you do.

#### 4. Finish the work — don't ping-pong

Excessive back-and-forth burns context + DB rows + reader attention. Anti-patterns:

- ❌ Sending "ok will do" before doing it (combine into one final reply)
- ❌ Asking clarifying questions one-at-a-time (batch them in a single message)
- ❌ Status-pinging mid-task ("still working...") — only message at start (claim) and end (result)
- ❌ Replying just to acknowledge without adding info (the `read_at` timestamp is the ack)

**Target: ≤ 2 messages per task** (1 claim + 1 result). 3+ rounds = re-scope or escalate to CEO.

#### 5. Surface parity (CC + Desktop + curl)

All three surfaces hit the same DB. Same rules apply to all:

| Surface | Send via | Inbox via |
|---------|----------|-----------|
| Claude Code (stdio MCP) | `mcp__mooniex-coord__send_message` | UserPromptSubmit hook (auto) |
| Claude Desktop (Remote HTTP MCP) | `mcp__MoonieX_MCP__send_message` (or whatever the connector exposes) | manual `read_inbox` at chat start |
| curl / scripts | POST `/api/office/messages/send` | GET `/api/office/messages/inbox?for_agent=...` |

The `from_agent_id` rule (#1) keeps audit clean across all surfaces.

### What NOT to do

- ❌ Spoof `from_agent_id`
- ❌ Send anonymously (no agent_id, or shared / generic id)
- ❌ Skip `read_inbox` at session start (Desktop / external) — leaves senders waiting
- ❌ Reply more than once per task without new info — context bloat
- ❌ Put secrets / tokens / PII in body or metadata
- ❌ Use messaging for state that belongs in DB (use desk claim, not "I'm at Desk-X" message)
- ❌ Auto-respond loops — every agent auto-replying = fork bomb
- ❌ Edit `webapp_office_messages` directly via SQL — go through API

Full playbook: [playbooks/agent-messaging.md](https://github.com/PASAKON/Agents-Wikis/blob/main/playbooks/agent-messaging.md) (`org:playbooks/agent-messaging.md`).

---

## Section 31 — One CTO per repo (parallel-CTO sharding) (CEO directive 2026-06-05)

Running 2–3 CTO sessions in parallel? **Each CTO owns one repo; never cross into another CTO's repo.** This is the simple fix for the two failure modes hit on 2026-06-05: (a) two CTOs merging the same branch (`bootstrap/landing-mvp`) → race / overwrite; (b) a CTO unable to merge another CTO's task because the branch + worktree live only in the other session (not pushed, not on this machine).

### 31.1 Default assignment (adjust as needed, but keep it 1:1)

| CTO | Repo(s) |
|-----|---------|
| CTO-1 | `mooniex-webapp` |
| CTO-2 | `mooniex-claudeflow` |
| CTO-3 | `warpclip-webapp` / `mooniex-claudesign` / `mooniex-claude-skills` |

### 31.2 Rules

1. A CTO only **creates / delegates / reviews / merges** tasks for **its own repo**. No exceptions.
2. **Never merge, hold-override, or push another CTO's task** — even if its DEV report relays into your session (the shared `state/tasks.db` broadcasts every report to all CTO tabs). Respect `owner_cto`.
3. The shared `state/tasks.db` is the **coordination layer**: read-only visibility into other CTOs' work. Do not act on what you don't own.
4. **Re-assigning a repo** to a different CTO → the previous owner pushes any in-flight branch to `origin` first, so the new owner can access it.

### 31.3 Why this rule exists

On 2026-06-05 a held, security-sensitive task (`task-d6943170`, CFO RLS/RBAC observability) carried a CTO "do NOT merge — acceptance unverified" note, yet a second CTO session merged + pushed it to prod before the RLS-403 / perf / cron acceptance was verified (`owner_cto` was `None`, and nothing stopped a cross-session merge). Separately, an event-page task (`task-237bfe8d`) could not be merged by the main CTO at all — its branch + worktree existed only in the other session, not on this machine. Sharding by repo eliminates both: one owner, one worktree, one merger per repo.



---

