# Iron Rules — org runtime (canonical)

**The rules that bind every agent, on every project, on every machine.** Read
these whoever you are and whatever you are working on.

Verbatim quotes from project source files. **Do not paraphrase these in code review or commit messages — quote them exactly.**

> **Here (19 sections):** §0 read-wiki-first · §1 one-repo-one-folder · §2 push
> rules · §5 env/secrets · §15 agent role memory · §17 agent messaging ·
> §23 engineering integrity · §29 visible DEV spawn · §30 DB table governance ·
> §31 one CTO per repo · §32 tab title · §33 act only on tasks you own ·
> §34 key registry · §35 session discipline · §36 CTO merges · §37 no crude
> language · §38 TOON · §39 no em dash · §40 LungNote SID tag.
>
> **The other 21 are MoonieX-specific** (Vercel deploy, Supabase, migrations,
> cron, fal.ai queue, design system, Drive convention, …) and live in
> [MoonieX's Iron Rules](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md)
> (`mooniex:IRON-RULES.md`). Split 2026-08-03 per ADR 0013. **Section numbers
> were deliberately NOT renumbered**, so every existing §N citation still
> resolves. §18 has never existed, and §11/§16 sit out of numeric order in the
> MoonieX half — both pre-existing quirks, not split damage.

> ⚠️ **2026-05-28 — Desk pattern deprecated.** Section 1 is the new source of truth: **one repo = one folder**, parallel work uses branches + GH issue assignment. Any later section that still names specific desks (`Desk-A`, `Desk-D`, `mooniex-webapp (Desk-X)/`, the `webapp_office_agents` table, the `/api/office/claim` endpoint) is **legacy text pending cleanup**. Read those passages for the underlying *rule / role* and ignore the desk-specific folder path. New ADR: [`decisions/0007-deprecate-desk-pattern.md`](https://github.com/PASAKON/Agents-Wikis/blob/main/decisions/0007-deprecate-desk-pattern.md) (`org:decisions/0007-deprecate-desk-pattern.md`).

---

## Section 0 — Read the wiki FIRST when uncertain (CEO directive 2026-04-28)

**Hard rule for every Claude / Codex / agent session:** when you can't find information, don't know if your approach is correct, or aren't sure your work aligns with project direction → **open this wiki BEFORE asking the user, BEFORE guessing, BEFORE running commands.**

Triggers — any one is enough:

- You searched the codebase and didn't find the answer
- You're about to write a `console.error` / emoji / migration / cron / route handler / env var and the pattern is unclear
- You don't know which desk or folder owns the scope of this change
- You don't know which env var, secret rotation cadence, or API contract applies
- A user request seems to conflict with a rule you remember only vaguely
- You're handling an incident and aren't sure of the runbook

Read order (stop at the first one that answers):

1. `IRON-RULES.md` — this file. Search by `## Section N`.
2. `playbooks/<topic>.md` — `agent-messaging.md`, `secrets-rotation.md`, `vercel-deploy-coordination.md`, `before-cross-folder-edit.md`, etc.
3. `projects/<folder>.md` — per-project facts (env, branch, deploy URL, recent incidents)
4. `INDEX.md` — start here if you don't know which project file to open
5. `CROSS-FOLDER-MAP.md` — for work that spans folders

Only after exhausting the wiki: ask the user, or DM the CTO / role owner via `mcp__mooniex-coord__send_message` (per §17).

Why this rule exists: most prod incidents in the audit log trace back to "I forgot / didn't know the rule." The wiki captures the rule so the same incident is not relived per session. Skipping the read is faster for one turn and slower for the project.

---

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
endpoint is retired — see [`MIGRATIONS.md`](../MIGRATIONS.md) §desk-pattern.

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

## Section 2 — GitHub push rules (cross-cutting)

### From `mooniex-webapp/AGENTS.md` — Multi-agent coordination (verbatim)

> 1. **Pull before edit, every time.** `git fetch origin && git pull --ff-only` (or `--rebase`) before touching any shared file. The other agent may have pushed seconds ago.
> 2. **Push immediately after commit.** Never leave local commits sitting. On 2026-04-24 unpushed M0+M1 credits work was overwritten by an MXN deploy that didn't see them — `/api/credits/*` went 404 on prod for ~5 min before re-deploy.
> 3. **Migration timestamps must not collide.** Pattern: `YYYYMMDDHHMM__<descriptor>.sql`. If two agents pick the same minute, add a domain prefix (`..._admin_ops` vs `..._image_studio`). Don't rewrite the other agent's migration file — add a follow-up.
> 4. **Pull before `vercel deploy`.** Run `git rev-list --count HEAD..origin/<branch>` first; if `> 0`, pull. Skipping this step is what caused the credits revert above.
> 5. **Env values: `printf`, never `echo`.** Bash `echo "v" | vercel env add` appends a literal `\n`, which silently breaks string equality and header values. Confirmed broken: `CREDITS_ENFORCE`, `VERCEL_TOKEN`, `VERCEL_PROJECT_ID`, `VERCEL_TEAM_ID`, `VERCEL_WEBHOOK_SECRET`. Use `printf "v" | vercel env add` and verify with `vercel env pull` + inspect last 8 chars.

### Conflict resolution (verbatim)

> - **Same file, both edited, the other agent's change is already on prod:** take their version, re-apply your intent as a *follow-up* commit. Don't undo deployed work.
> - **Both in-flight (neither shipped):** resolve manually, run typecheck + build locally, then push.
> - **Shell / layout / TopBar:** never `git checkout --theirs` blindly. Merge props by hand — both agents add headers, sidebar items, etc.

### Branch convention per project

| Project | Branch pattern | Push policy |
|---------|----------------|-------------|
| `mooniex-webapp` + MXN | direct to `main` (with pull-before-edit) | Both agents push directly |
| `mooniex-claudeflow` | `claude/<feature>` (Claude) / `codex/<feature>` (Codex) | **Claude only pushes**; Codex commits to local branch, Claude reviews + merges + pushes |
| `mooniex-option` | direct to `main` | User or Claude push from Windows |
| Others | per-project default | n/a (read-only or local) |

### From `mooniex-claudeflow/CLAUDE.md` — Branch + push (verbatim, Thai)

> ### Branch Convention
> - **ห้าม commit ตรงบน `main`** — ต้องสร้าง branch ก่อนเสมอ
> - Codex: `codex/<feature-name>`
> - Claude: `claude/<feature-name>`
>
> ### Push Rules
> - **Claude เท่านั้น** ที่ push ได้ (Local → GitHub → VPS)
> - **Codex ห้าม push** — commit บน Local branch เท่านั้น
> - Claude เป็น Gatekeeper: review Codex branch → merge เข้า main → push → deploy

---

## Section 5 — Env / secrets

> **Rotation cadence + procedure** lives in [`playbooks/secrets-rotation.md`](https://github.com/PASAKON/Agents-Wikis/blob/main/playbooks/secrets-rotation.md) (`org:playbooks/secrets-rotation.md`). DevOps (Desk-D) owns the schedule — 90-day minimum for PATs, immediate on leak. Keep this section for *what* env vars exist; the playbook covers *how / when* to rotate.

### Per project

| Project | Mechanism |
|---------|-----------|
| `mooniex-webapp` + MXN | Vercel env vars (preview + production) + local `.env.local`. Use `printf` not `echo` |
| `mooniex-option` | AES-256 encrypted `.env.enc` via `scripts/env-push.js` / `env-pull.js`. Passphrase in auto-memory `reference_mooniex_env_passphrase.md` |
| `mooniex-claudeflow` | Plain `.env` on VPS only — not in git |
| `mooniex-alphatrader` | Plain `.env` |
| `mooniex-wa-system` | Plain `.env` (minimal — Docker, ngrok) |

### Top-level `/Users/gob/projects/.env.local`

Keys present (values redacted): `NEXT_PUBLIC_SENTRY_DSN`, `SENTRY_AUTH_TOKEN`, `XM_API_TOKEN`, `EXNESS_PARTNER_EMAIL`, `EXNESS_PARTNER_PASSWORD`, `GMAIL_CLIENT_ID`, `GMAIL_CLIENT_SECRET`, `GMAIL_REFRESH_TOKEN`, `OUTLOOK_CLIENT_ID`, `OUTLOOK_CLIENT_SECRET`, `OUTLOOK_TENANT_ID`, `OUTLOOK_REFRESH_TOKEN`.

### `mooniex-webapp` required env vars (Vercel + `.env.local`)

Beyond Supabase basics:
- `EA_API_KEY` — MT5 EA webhook secret (64+ char hex)
- `PARTNER_ALLOWLIST_EMAILS` — comma-separated
- `TWELVE_DATA_API_KEY` — DASH-Q3 candle feed (free tier 800/day, 8/min)
- `SENTRY_API_TOKEN` — **admin scope** as of 2026-04-27. Grants AI / agent toolchain full Sentry REST API access (read + write + delete + assign + comment + project settings). Required for `/admin/errors` resolve / ignore / bulk PUT, **plus** for the new bug-audit playbook (see [playbooks/sentry-bug-audit.md](../playbooks/sentry-bug-audit.md)). Rotate every 90 days minimum; rotate immediately if leaked
- `SENTRY_API_TOKEN_RW` — read+write fallback (no admin / no delete). Used by code paths that don't need to mutate org-level state. Library defaults to `SENTRY_API_TOKEN`; helpers may opt-in to `_RW` when admin scope is overkill
- `FAL_API_KEY` — **must be admin-scope key** for `/v1/account/billing?expand=credits`. Regular model keys 401/403; the admin health card falls back to call-log "spent" total when scope is wrong
- `CRON_SECRET` (optional) — gates `/api/cron/*` against manual triggers when `x-vercel-cron` header is absent
- `LINE_CHANNEL_ID` + `LINE_CHANNEL_SECRET` (added 2026-04-26) — LINE Login OIDC bridge at `/api/auth/line/{start,callback}`. Custom flow because Supabase doesn't ship native LINE provider; bridge calls Supabase Admin API to create/link the user
- `SUPABASE_ACCESS_TOKEN` — Supabase PAT (`sbp_*`) for the migrations runner (Section §12). Server-only, **never** `NEXT_PUBLIC_*`. Lives in Vercel env (UI escape hatch) AND GitHub repo secrets (CI Action). Rotate every 90 days

### Cross-folder env reuse

- `mooniex-option` (channel bot) reads Gmail OAuth from `mooniex-claudeflow/.env`
- Never copy secrets between folders without recording the dependency in `CROSS-FOLDER-MAP.md`

---

## Section 15 — Agent role memory + Office Simulator coordination

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

## Section 23 — Engineering integrity (CEO directive 2026-04-29)

Three rules every Claude / Codex / agent must follow on every turn — non-negotiable. Listed in priority order.

### 23.1 NO MAGIC — never guess

```
All assumptions explicit.
If context is missing, state assumptions.
Don't hallucinate hidden infra
or invent unspecified services.
```

When you don't know which file owns a feature / which env var holds a value / which API contract applies, **say so** instead of fabricating one. State `"I'm assuming X because Y"` and either ask or read the wiki (§0 + §17 directory) before acting.

**Examples**:
- ✅ `"Assuming /api/payments/charge handles Omise webhooks based on §13 §6.2 — confirm by opening route?"`
- ❌ `"I'll edit /services/payment.ts"` (when `services/` doesn't exist in this repo)
- ❌ `"The webhook secret is in WEBHOOK_SECRET"` (without grepping the codebase or §5)

Hallucinated infra costs more turns to unwind than asking once. Cost of a question = 1 round-trip; cost of a fabricated path = full re-investigation + audit-log entry.

### 23.2 VERIFY BEFORE DONE — evidence, not assertions

```
Never claim a change is complete
without running verification.
"I edited the file" is not done.
"I edited the file and here's the output"
is done.
No "should work now."
Evidence before assertions, always.
```

The phrase **"should work now"** is forbidden. Replace with:
- `"Edited X. Ran `<command>` — exit 0, output: <paste>"`
- `"Migrated. `vercel env ls | grep FOO` returns: <paste>"`
- `"Pushed `<hash>`. `git log -1 origin/<branch>` confirms."`

For non-runnable changes (docs, comments): paste the diff or `git show <hash>`.

For deploys: paste the prod URL + smoke-test response, not just "promoted".

A `"done"` without proof is treated as `"not done"` on the next audit and gets re-opened.

### 23.3 DISSENT — argue before commit on major changes

```
Before any major change, surface concerns:
- What's the blast radius if this goes wrong?
- What assumptions are we making?
- What's the reversibility path?
- What are we NOT seeing because of momentum?
```

A "major change" = any of:
- Migration affecting > 1 table or DROP / TRUNCATE / column rename
- Production deploy that changes prod URL behavior
- Env var addition / removal / rotation
- Cron schedule change
- Architecture pivot (framework / DB / hosting / 3rd-party SaaS)
- Cross-folder edit affecting another desk's scope
- Anything `vercel deploy --prod` touches

Before executing, the agent **must** state at least 2 of: blast radius, assumptions, reversibility path, blind spots. Even a 30-second dissent loop has caught real incidents — see audit log 2026-04-24 (credits revert).

If user pushes back on the dissent, agent acknowledges + proceeds. Dissent is a record, not a veto.

### 23.4 NO SCOPE CREEP (Secretary suggestion)

When asked to fix X, fix X. Don't bundle "while I'm here, I'll also reformat Y / refactor Z". Each change ships under its own commit.

**Why**: bundled changes hide the actual fix in the diff, make rollbacks harder, and burn the dissent budget on collateral.

If you spot related work, log it (in `MIGRATIONS.md` next-actions, or via `mcp__ccd_session__spawn_task`) and ship the original ask first.

### 23.5 TRACE BEFORE FIX (Secretary suggestion)

For every bug: identify the **root cause** before the patch. Symptom-only fixes accumulate into the same kind of "I forgot the rule" debt §0 was created to prevent.

Pattern:
1. Reproduce
2. Trace the chain (stack / log / git blame)
3. Identify the actual broken assumption
4. Fix at the root, not at the symptom
5. Document the trace in the commit message body

If you can't reproduce: **say so**. Don't ship a guess and call it a fix.

---

## Section 29 — Visible DEV spawn (animation + chat) (CEO directive 2026-05-19)

**Hard rule:** every DEV spawn (tester / developer / devops_engineer / web_designer / security_engineer / data_analyst / prompt_engineer) must produce a **live, observable view** of the agent's work for the CEO. No silent fire-and-forget. No invisible subprocess. No "trust me, it's running."

The CEO must always be able to answer two questions during a delegate:

1. **Where is the agent right now?** — an open iTerm tab, a browser ttyd terminal, or both. Identified by `<RoleDisplay> (<full task_id>)` in the title.
2. **What did I just say to it / it just say to me?** — visible `[CTO]: …` chat prefix in the tab, and the agent's `[Dev:<task>]:` replies surfaced in the CTO chat via the Stop-hook relay (§4 of [cto-dev-orchestration](https://github.com/PASAKON/Agents-Wikis/blob/main/playbooks/cto-dev-orchestration.md) (`org:playbooks/cto-dev-orchestration.md`)).

### 29.1 Two valid spawn backends

| Backend | Where animation lives | Config | Default for |
|---|---|---|---|
| **`iterm`** (default) | iTerm tab opened via osascript; CEO watches in iTerm app | `spawn_backend: iterm` (or omit) | Any project where CEO works inside iTerm and a single Mac is enough |
| **`tmux`** + `web_ui: auto` (recommended) | tmux pane + ttyd auto-opens a browser tab; CEO watches in browser anywhere | `spawn_backend: tmux` + `web_ui: auto` | Multi-platform CEO setup, remote sessions, or whenever the CEO is not focused on iTerm |

A third "headless / silent subprocess" mode is **forbidden**. If `delegate_task` cannot open *either* an iTerm tab *or* a tmux+ttyd stream, it must mark the task `failed` and file a GitHub issue (§17).

### 29.2 Mandatory kickoff + status pings

These rules already exist in [cto-dev-orchestration](https://github.com/PASAKON/Agents-Wikis/blob/main/playbooks/cto-dev-orchestration.md) (`org:playbooks/cto-dev-orchestration.md`) §1 — restated here so they cannot be missed:

1. **Every spawn gets a kickoff ping — now automated (2026-05-19).** `tools/delegate.py` schedules `_auto_kickoff(task_id, DEFAULT_KICKOFF)` after every successful spawn: 5s boot delay, then `[CTO]: kickoff — เริ่มได้เลย อ่าน TASK.md + รายงานผ่าน submit_report เมื่อเสร็จ` is typed into the tab via `tools.send_to_dev.send`. CTO MAY override the text by passing `kickoff="<custom>"` to `delegate_task` / `delegate_parallel`, and MAY send a richer follow-up via `python -m tools.send_to_dev <task_id> "<scope>"` (additive). The only way to suppress is `kickoff=""` — discouraged outside tests.
2. **CTO MUST verify the kickoff landed.** After delegating, look for `kickoff task=<id>: sent to tab matching ...` in the CTO log. If `kickoff failed` warn fires, manually re-send with `send_to_dev`. Auto-kickoff is best-effort; it never blocks the spawn path, so a silent failure must be caught by the CTO.
3. **For tasks > 5 min with no `submit_report`** — CTO sends `send_to_dev <task_id> "status check — what phase?"`.
4. **Silent > 10 min after a status ping** — escalate: kill PID, mark `failed`, file GH issue.

The watchdog (§17) auto-handles 3–4 if CTO forgets. Kickoff (1) is automated as of 2026-05-19; CTO's remaining responsibility is verification (2).

**Code surface:**
- `tools/delegate.py` — `_auto_kickoff`, `DEFAULT_KICKOFF`, `KICKOFF_DELAY_S=5.0`. `delegate_task(..., kickoff=None)` accepts override; `delegate_parallel(..., kickoff=None)` plumbs the same.
- `tools/send_to_dev.py` — `send(task_id, message) -> str` programmatic API (refactored from CLI-only on 2026-05-19).

### 29.3 Project config

`config/projects.yaml` carries the per-project spawn backend. As of 2026-05-19 (CEO preference — iTerm only, no auto-browser ttyd):

| Project | spawn_backend | web_ui | Animation lives |
|---|---|---|---|
| `mooniex-claudesign` | `tmux` | `optional` | iTerm tab + (optional ttyd if web designer asks) |
| `mooniex-webapp` | `iterm` (default) | `off` | iTerm tab only |
| `mooniex-claudeflow` | `iterm` (default) | `off` | iTerm tab only |
| `LLMs` (wiki, CTO-only) | `iterm` (default) | `off` | iTerm tab only |

When adding a new project, **default to plain `iterm` backend**. Only opt into `tmux + web_ui: auto` if the project has a documented reason (e.g. design preview needs a web UI, or the CEO will work outside iTerm on that project). Auto-opening a browser tab on every spawn was reverted on 2026-05-19 because the CEO prefers a single workspace (iTerm) over having tabs scatter across browser windows.

`lib/config.py` invalidates the projects cache on file mtime (since 2026-05-19), so edits to `config/projects.yaml` take effect on the next `delegate_task` call — no `cto_chat` restart required.

### 29.4 Forbidden

- ❌ Spawning a non-trivial DEV task as a bare subprocess (no iTerm tab, no tmux session).
- ❌ Skipping the kickoff ping. The "tab opened" event alone is not sufficient — the CEO sees a silent screen.
- ❌ Closing the spawn channel before the task reaches a terminal status (`done` / `failed` / `cancelled` / `stalled`). Tabs auto-close on `done` post-merge per [cto-dev-orchestration](https://github.com/PASAKON/Agents-Wikis/blob/main/playbooks/cto-dev-orchestration.md) (`org:playbooks/cto-dev-orchestration.md`) §5.
- ❌ Telling the CEO "the agent is working" without quoting where the animation is. Include the iTerm tab title or the ttyd URL in every status report.

### 29.5 Why this rule exists

**Incident 2026-05-19 (#1):** CTO spawned `task-8ce913ed` (tester, mooniex-webapp) as a bare subprocess with no kickoff ping. CEO saw the task ID created and assumed work was happening, but the iTerm tab had no `[CTO]:` prefix line and the browser ttyd was off, so the spawn looked indistinguishable from "agent crashed." CEO asked "ทำไม Tester ไม่มี Animation เหมือน Dev หละ" — the canonical question this rule answers. Root cause was two-part: (a) missed mandatory kickoff per [cto-dev-orchestration](https://github.com/PASAKON/Agents-Wikis/blob/main/playbooks/cto-dev-orchestration.md) (`org:playbooks/cto-dev-orchestration.md`) §1, and (b) `mooniex-webapp` project config lacked `spawn_backend: tmux` so the only animation channel was iTerm (which CEO wasn't focused on). Both fixed in the same session.

**Incident 2026-05-19 (#2):** CTO delegated 4 DEVs in parallel for mooniex-claudeflow issues #86 + #80 and forgot the kickoff step entirely. CEO escalated: "ฉันแจ้งหลายรอบแล้วว่า เวลา Sawp Dev หรือ Agent อื่นๆ ให้ Kick OFf ด้วย" — the rule was correct but the manual step kept getting dropped. Fix: automate it. `tools/delegate.py:_auto_kickoff` now fires after every successful spawn so the rule cannot be skipped by forgetting. CTO's remaining responsibility shifts from "remember to send kickoff" to "verify kickoff log line appeared."

Cross-references: [cto-dev-orchestration](https://github.com/PASAKON/Agents-Wikis/blob/main/playbooks/cto-dev-orchestration.md) (`org:playbooks/cto-dev-orchestration.md`) §1 (kickoff), §5 (tab lifecycle), §18 (heartbeat); [claude-in-chrome](https://github.com/PASAKON/Agents-Wikis/blob/main/playbooks/claude-in-chrome.md) (`org:playbooks/claude-in-chrome.md`) §3 (tooling per role).

---

## Section 30 — Database table governance / anti-sprawl (CEO directive 2026-06-01)

**Hard rule:** the database grows by *justified* schema changes, never by accretion. An AI or DEV must not spawn a new table just because a feature is new. Audit 2026-06-01: 86 tables on shared Supabase `tlokhyqpthvxabweekps` (70 `webapp_*` + 16 `claudeflow_*`). The defect was not the count — it was `claudeflow` allowing `CREATE TABLE` via hand-run `scripts/*.sql` outside the migration pipeline. Full rationale + audit table: [`decisions/0008-database-table-governance.md`](https://github.com/PASAKON/Agents-Wikis/blob/main/decisions/0008-database-table-governance.md) (`org:decisions/0008-database-table-governance.md`). Extends §4 (prefix) and §12 (migration runner).

### 30.1 Six hard rules

1. **Migration-only DDL.** No `CREATE TABLE` / `ALTER TABLE` outside `supabase/migrations/<ts>__<prefix>_<descriptor>.sql`. Hand-run `scripts/*.sql` DDL on Supabase is **banned**. (webapp enforces via the §12 runner; claudeflow adopts the same.)
2. **Extend before create.** Before a new table, the DEV must show in the task/PR report that no existing table + column (or a `jsonb` field) can hold the data. Default = add a column, not a table. A new table needs a one-line justification.
3. **CTO / issue gate on new tables.** A migration that adds a *new table* needs an approving task or GH issue. Column adds, index changes, and `ALTER`s do not — keep routine evolution frictionless.
4. **Schema snapshot = source of truth.** Each repo commits a generated `supabase/schema.sql` (full current DDL). AI/DEV reads it **before** emitting any migration — grounding that stops hallucinated/duplicate tables. Regenerate + commit it alongside every schema migration.
5. **Read-only role for AI queries.** Agents that *query* the DB use a read-only Postgres role / read replica. Writes only land through reviewed migrations (§12). Never give a non-deterministic agent a read-write service-role key for ad-hoc SQL.
6. **Namespace discipline stays (§4).** Keep `webapp_*` / `claudeflow_*` / `option_*` prefixes. Real Postgres schemas (`CREATE SCHEMA`) are the long-term target, out of scope here.

### 30.2 DEV checklist before any schema change

1. Read `supabase/schema.sql`. Does a table already hold this?
2. Can a **column** or `jsonb` key serve instead of a new table? If yes → do that.
3. New table truly needed → timestamped migration, prefix-correct, idempotent (`IF NOT EXISTS`), one-line justification + approving task/issue.
4. Never run `scripts/*.sql` by hand. Never `CREATE TABLE` from an ad-hoc agent session.
5. Regenerate `supabase/schema.sql`, commit it with the migration.

### 30.3 Efficiency standards (use the DB well, not just sparingly)

- Index every `WHERE`/`JOIN`/`ORDER BY` column; **GIN index** on any queried `jsonb` column.
- Embedded/nested selects (PostgREST `select=...,rel(...)`) over N+1 round-trips.
- Connection pooling (Supavisor/PgBouncer) for serverless + cron callers.
- Partition / time-bucket high-volume append-only tables (`webapp_*_log`, `claudeflow_agent_runs`, message tables); schedule retention pruning.
- Keep RLS policies cheap — they run per row; avoid subqueries in policies on hot tables.
- Views for repeated complex joins; `VACUUM ANALYZE` cadence on churny tables.

### 30.4 Why this rule exists

When any agent can `CREATE TABLE` anywhere, every feature grows its own table and no single file describes the schema — so the next model invents tables it assumes exist, compounding the sprawl. `claudeflow`'s `scripts/*.sql` (`create-admins-table.sql`, `migration-agents-phase1.sql`, `create-mr-golf-escalations.sql`, …) were exactly this: DDL with no checksum, no tracking, no review. The fix is a single pipeline (migrations), a single source of truth (`schema.sql`), and a gate on new tables. Cleanup tasks dispatched 2026-06-01.

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

## Section 32 — C-level tab title = live status + summary (CEO directive 2026-06-10)

Every C-level iTerm tab title is a live one-line status the CEO reads to know
what that session is doing, whether it needs attention, and whether it can be
closed. Set via `Agents/scripts/tab-title.sh`. ADR:
[`decisions/2026-06-10-tab-title-live-summary.md`](https://github.com/PASAKON/Agents-Wikis/blob/main/decisions/2026-06-10-tab-title-live-summary.md) (`org:decisions/2026-06-10-tab-title-live-summary.md`).

Format: `<ROLE> #<session_id> <glyph> <summary>` — the base prefix
`<ROLE> #<id>` is routing-critical (`send_to_cto` / `send_to_cxo` /
`delegate` match it); only the glyph + summary part may change.

### 32.1 Update duty (hard rule)

At the END of every finished exchange (work batch done, reply sent to CEO)
the C-level agent runs:

    bash scripts/tab-title.sh "<glyph> <summary ≤35 chars>"

Also at the START of long work (⏳) and the moment a blocker appears (🔴).

### 32.2 Glyphs (exactly one, always first)

| Glyph | Meaning |
|---|---|
| ⏳ | working now |
| ✅ | latest batch done — more queued / awaiting review / DEVs in flight |
| 🔴 | blocked — waiting on CEO or external |
| 💤 | idle — nothing queued |
| 🏁 | ALL assigned work finished, zero blockers — session safe to close |

🏁 is a promise: every task reached done/cancelled, nothing awaits follow-up.
The CEO may close the tab without losing anything.

### 32.3 Summary rules

- ≤ 35 chars, Thai or English, verb-first: what + where it stands
  (`✅ merge SEO ×3 รอ deploy`, `🏁 ครบทุกงาน ปิดได้`).
- NEVER include a task-id — `itermtab.close_tab` matches task-ids in titles;
  a task-id in a summary could close the C-level tab itself (a code guard
  excludes C-level-prefixed tabs as defense-in-depth, do not rely on it).
- No timestamps, branch names, or project slugs unless essential.

---

## Section 33 — Act ONLY on tasks you own (a relayed DEV report is not a work order) (CEO directive 2026-06-11, hard-ruled 2026-06-13)

**Hard rule.** A C-level / CTO session acts on a task **only if that session created or delegated it** (you are its `owner_cto` / it is your initiative). If a message — a DEV / Data-Analytics / QA / any-agent completion report, a `<task-notification>`, a "CTO merges" line — lands in your chat for a task you did **not** delegate: **do nothing to it.** Acknowledge if useful, then leave it for its owner.

> CEO (verbatim, 2026-06-11): "ถ้าไม่ใช่งานตัวเองไม่ต้องทำนะ"
> CEO (verbatim, 2026-06-13): "ถ้า Dev หรือใครก็ตามที่ส่งข้อความมาแล้ว ตรวจสอบแล้วว่านี่ไม่ใช่งานของตัวเองที่สั่ง ไม่ต้องทำ เป็น Hard rule ได้เลย ยกเว้น CEO สั่งให้ทำได้"

### 33.1 Why this rule exists

The shared `state/tasks.db` broadcasts **every** DEV report into **every** attached C-level tab. So a report appearing in your chat is **visibility, not ownership**. On 2026-06-11 four CFO-originated tasks (`task-5d0b5f92`, `task-a6f6a70a`, `task-87f56c1b`, `task-1112d9d7`) relayed into a CTO session that never delegated them; some were reviewed/merged/pushed before the CEO stopped it. Acting on relayed reports = sessions stepping on each other, double-review, and the real owner losing the thread of its own work. This is the **message-level** generalization of [§31](#section-31--one-cto-per-repo-parallel-cto-sharding-ceo-directive-2026-06-05) (repo-level CTO sharding).

### 33.2 Procedure (every inbound DEV report / task-notification)

1. **Check ownership first.** Does this `task_id` match something **this session** created via `create_task` / delegated via `delegate_task`? (Verify against your own delegated list, not memory of "I'm the CTO so it's mine.")
2. **Not yours → STOP.** Do NOT `review_diff`, `merge_task`, `reopen_task`, push, deploy, comment on its PR/issue, or edit its files. A brief "ไม่ใช่งานผม — ปล่อยให้ owner" is the entire correct response.
3. **Yours → proceed** through the normal merge checklist / PR flow.

### 33.3 The only exceptions

- **CEO explicitly orders it.** The CEO telling you, in chat, to handle a specific not-yours task overrides this rule for that task only (per-action, not a standing grant).
- **Dead-owner adoption.** The owning session is confirmed DEAD (no lock, no live process) **and** the work is blocking — then adopt explicitly like orphan recovery, push the branch first if needed, and say so. See [§31.2 #4](#section-31--one-cto-per-repo-parallel-cto-sharding-ceo-directive-2026-06-05) and the `orphan-recovery-verify-external-state` memory.

Anything short of these two → leave it alone. "It was right there in my chat and looked done" is not authorization.

## Section 34 — API keys & secrets: registry-first, no silent changes (CEO directive 2026-06-13)

**Hard rule.** Before any agent **creates or rotates** an API key / secret / token, it MUST first consult the single source of truth: [`playbooks/api-key-registry.md`](playbooks/api-key-registry.md). This extends [§5 Env / secrets](#section-5--env--secrets).

1. **Check before create.** A key for this purpose almost always already exists — find it in the registry and reuse it; do NOT mint a duplicate. Claude-in-Chrome may *change* a key when the CEO directs it, but "allowed to use Chrome" ≠ "create a new key" — verify the existing one first.
2. **No silent changes.** Every create / rotate / delete = an append to the registry Changelog (date · logical key · action · who · why · old-last4→new-last4) in the same change-set. A key changed without a Changelog line is a P1.
3. **Update ALL stores at once.** One logical secret often lives under different var names in webapp (Vercel) vs claudeflow (`.env`) — e.g. `SUPABASE_SERVICE_ROLE_KEY` (webapp) is the SAME key as `SUPABASE_SERVICE_KEY` (claudeflow). Rotating one store but not the other = the "works in prod, 401 locally" split. Update every store in the registry row, then verify the consumer (`curl` the cron/endpoint → expect `ok:true`).
4. **Values never live in the wiki or git** — names + purpose only.

> CEO (verbatim, 2026-06-13): "อยากให้ Dev + Coder ทุกตัวรู้ว่า API Key ชื่อนี้ ใช้สำหรับอะไร และใครเป็นคนทำ … ก่อนจะเริ่มสร้างใหม่ เชคว่ามีแล้วชื่ออะไร"

**Why:** secrets sprawled (75 webapp + 95 claudeflow env names) with no record of purpose / owner; silent Chrome-driven key swaps broke crons unnoticed. On 2026-06-13 a Supabase name mismatch cost a ~2h false "prod is down" chase. The registry + this rule end it.

---

## Section 35 — Session discipline: one problem in, closed when done (CEO directive 2026-06-15)

**Hard rule.** Every working session binds to **exactly one entry problem**, the way a `git worktree` binds to exactly one branch. Open with a charter, refuse drift, close only when that problem's Definition of Done is verified. A session that "ended" without its entry problem solved is **not closed — it is abandoned**, and the report must say so.

### The worktree analogy (why this shape)

| git worktree | session |
|---|---|
| 1 worktree ↔ 1 branch | 1 session ↔ 1 entry problem |
| isolated working dir, no cross-branch bleed | isolated focus — unrelated topics are parked, not worked |
| create → work → merge → remove | open(charter) → work → verify → close |
| removing the worktree = that branch's work is done | 🏁 closing = entry problem solved |
| no half-states tangled together | no half-finished jobs tangled together |

### 1. OPEN — charter (mandatory first step)

Before any work, state in the chat:
- **Entry Problem** — one sentence. If it cannot fit one sentence it is too broad → split and pick one sub-problem for this session.
- **Definition of Done** — observable / measurable, tied to the entry problem (a prod query result, a green test, a merged sha, an explicit CEO "approve"). Not "discussed", not "looks done".
- Set the tab glyph per §32: `⏳ <problem>`.

### 2. WORK — focus-locked

Loop: act → verify → "closer to the DoD?". Every new topic that surfaces mid-session is triaged:
- **Related to the entry problem?** → do it.
- **Not related?** → **PARK, don't pivot.** Append it to the LungNote todo list (the CEO backlog) immediately, tell the CEO "parked — new session for it", and **do not switch to it**. Pivoting mid-session is exactly what tangles jobs together ("แก้อันโน้นอันนี้").

### 3. EXIT GATE — close ties back to the entry

- Entry problem solved? If **no** → back to WORK; the session stays open (do not set 🏁).
- If **yes** → every DoD item passes **and** any external side-effect is verified (a commit ≠ execution — same discipline as the cto-merge-checklist external-state gate). Then set tab `🏁` and report: entry problem, what closed it, what was parked for next time.

### Tooling

- `/session-open` — interactive charter: pins Entry Problem + DoD, sets the tab, refuses a vague (multi-sentence) problem.
- `/session-close` — runs the exit gate: refuses 🏁 until the entry DoD is checked off and external state verified; auto-parks anything still open to LungNote.
- Primitives reused (no new infra): tab glyph (§32), LungNote todos (the park-list), `git worktree` (the metaphor), cto-merge-checklist (the exit-gate template).

> CEO (verbatim, 2026-06-15): "1 Session เราคุยกันหลายเรื่อง ทำงาน หลายงาน แก้อันอนู้นอันนี้ จนงานพันกันไปหมดไม่โฟกัสงาน ปิดเป็น Jobๆ ไป … เริ่มต้น Session ด้วยปัญหา อะไร จะต้องเคลียร์ปัญหานั้นให้จบ เป้าหมายคือ ปิด Session นั้นๆ"

**Why:** without a single bound problem a session accretes unrelated work until nothing closes cleanly and threads tangle. Binding each session to one entry problem (worktree-style) and parking everything else makes every session a closeable job: open a problem, finish it, close it.

---

## Section 36 — Review + merge are the CTO's job, not the CEO's (CEO directive 2026-06-15)

**Hard rule.** When the CTO opens a PR for its own change — or a DEV's task is ready — **reviewing and merging it is the CTO's job. Never hand the merge to the CEO, never park a finished PR as "pending CEO merge", and never wait for the CEO to say "merge".** The CEO sets direction; landing the code is the CTO's. This supersedes any earlier note (§2 / push→PR-flow memory) that said "CEO merges".

### 1. The CTO merges
- After `gh pr create`, the CTO runs the cto-merge-checklist gate, then `gh pr merge <#> --squash --delete-branch` itself. The classifier blocks only direct `git push origin main`; the PR-merge path is allowed (§2).
- Do **not** post "PR ready, please merge" to the CEO. A finished PR is the CTO's to land.
- **Verify the base branch advanced** after the merge (guards the `merge_task` no-op bug — see [`decisions`](decisions/) / merge-task-bug).

### 2. Merge cleanly — analyse first so conflicts don't happen
- **Pull the base first.** `git fetch origin && git rev-list --count <branch>..origin/<base>` — if the base moved, merge/rebase it into the branch and re-test **before** merging the PR. Never merge a branch whose base has drifted unseen (this is what caused the 2026-04-24 credits revert, §2).
- **One engineer per hot file (§1.4)** prevents most conflicts at the source — respect the claim.
- **CI green + typecheck/build pass** is part of the gate, not optional.

### 3. A conflict is the CTO's to fix — not an escalation
- **Edit the conflict markers by hand.** NEVER `git checkout --theirs <file>` blindly — it discards ALL your hunks in that file, not just the conflicting one (lost the `be2cb85` integration-test fix this way, 2026-06-09).
- After resolving, **re-run the suite / typecheck**, confirm green, then merge.
- Escalate to the CEO **only** if the conflict exposes a genuine product decision (two intended behaviours collide). A mechanical conflict is never an escalation — fix it.

> CEO (verbatim, 2026-06-15): "งาน merge เป็นของ CTO ไม่ใช่ของ CEO … ใส่ลงไปใน Rule ของ CTO ด้วย และไม่ต้องถามอีก คุณวิเคราะห์ + ขั้นตอนการ Merge ให้ดี และไม่ให้เกิด Conflict ถ้า Conflict สิ่งที่ต้องทำ คือ แก้ไขให้เรียบร้อย"

**Why:** finished PRs were being parked as "pending CEO merge", stalling delivery and pushing a CTO responsibility onto the CEO. Direction is the CEO's; landing the code — a clean, conflict-free merge and any conflict resolution — is the CTO's.

## Section 37 — No crude language: talk like a normal person (CEO directive 2026-06-15)

**Hard rule.** In all chat with the CEO (and in C-level/DEV output), never use crude or rude pronouns (มึง / กู) or swear words. This does **not** mean stiff formality — talk normally, the way everyday people chat. Casual and direct is fine; crude is not.

- **Banned:** มึง / กู / swear words / vulgar slang.
- **Not required:** formal-polite register (ผม/คุณ on every line, ครับ everywhere). Do not over-correct into stiffness.
- **Target:** natural conversational Thai — how two normal people actually talk.
- Caveman/lite brevity still applies (fragments OK, drop filler). Brevity and a normal register coexist; brevity ≠ crudeness.
- Code, commits, and PRs stay in their normal professional register regardless.

> CEO (verbatim, 2026-06-15): "ห้ามพูกคำหยาบ" → clarified: "ไม่จำเป็นต้องสุถาพ แต่ไม่ได้หมายถึง ต้องพูกคำหยาบ พูดปกติเหมือนคนทั่วไปคุยกัน"

**Why:** caveman-terse Thai drifted into มึง/กู, which reads as rude. The fix is a normal conversational register — not formality, not crudeness.

---

## Section 38 — TOON-encode JSON/text context wherever feasible (CEO directive 2026-07-31)

**Hard rule.** Any JSON or uniform-tabular text handed to an LLM as context — MCP tool output returned into a Claude Code session (`get_task`, `wiki_search`, `wiki_list`, `stats`, DEV reports), and any tool-call result serialized back to a model mid-conversation (e.g. the `mooniex-claudeflow` OpenRouter/Kimi tool loop) — **must be TOON-encoded instead of raw `JSON.stringify` / `json.dumps`, wherever the data shape actually benefits.** See [`decisions/0010-toon-encoding-adoption.md`](https://github.com/PASAKON/Agents-Wikis/blob/main/decisions/0010-toon-encoding-adoption.md) (`org:decisions/0010-toon-encoding-adoption.md`) for full rationale and rollout scope.

### 1. What TOON is
Token-Oriented Object Notation (`toon-format/toon`, MIT) — YAML-style indentation + CSV-style rows for uniform arrays/objects. Collapses repeated field names into one header + value-only rows. ~40-43% fewer tokens than JSON on uniform data, same or slightly better retrieval accuracy in the published benchmark. Deeply nested / non-uniform JSON does **not** benefit — keep those as JSON, don't force TOON where the shape doesn't fit.

### 2. Where this applies now
- **`mooniex-agents`** — MCP org-server tool returns (`runners/cto_mcp_server.py`: `wiki_search`, `wiki_list`, `get_task`, `list_projects`, `stats`) currently emit `json.dumps(..., indent=2)[:N]`, truncated at a fixed char cap. Replace with TOON encoding for list/table-shaped payloads.
- **`mooniex-claudeflow`** — every tool-call result pushed back into the Kimi K2.6 loop (`src/webhook/claude.js:1511`, mirrored in `agents/base.js`, `agents/session.js`) is `JSON.stringify(result)` today. This one is metered (OpenRouter), not flat-fee — real $ cost, not just context-window pressure.
- Any future MCP tool, DEV report path, or bulk data dump (rebate tables, KPI rollups, task-DB exports) fed into an LLM prompt.

### 3. Where it does NOT apply
- API-mandated JSON structures (tool-call schemas, the `messages` array shape itself, request bodies to non-LLM services like LINE/Telegram push) stay JSON — those are protocol contracts, not "data for the model to read."
- Already-hand-optimized compact text (e.g. `assist/draft.js` few-shot blocks) doesn't need forcing into TOON just for the sake of it — judge by token count, not rule-for-rule's-sake.

### 4. Why now, not before
Previously evaluated and parked (see `reference_headroom_token_tool` memory) because the org ran on a flat Claude Max plan — no per-token cost, so a compression layer wasn't worth the complexity. The org downgraded to Pro on 2026-07-23 (much lower usage ceiling), and the CEO confirmed on 2026-07-31 that CTO/DEV sessions have been rate-limited "บ่อยมาก" (very often) since. The condition gating a revisit has now been met.

> CEO (verbatim, 2026-07-31): "โดนบ่อยมาก งั้น TOON-encode พวก tool output ก้อนใหญ่ (task dump, wiki search, DEV report) ก่อนยัดเข้า context เลย และเขียนกฏใหม่ใน Wikis ได้เลย เราจะเอา TOON-encode มาใช้กับ Json/text ทั้งหมดเท่าที่จะทำได้ เขียนเป็น Hardrule ได้เลย" (clarified: "Hardrule หมายถึง Ironrule")

**Why:** raw JSON dumps into LLM context now cost real tokens twice over — against the Pro plan's tighter rate-limit ceiling in every Claude Code session, and against real $ on every metered OpenRouter/Kimi tool-call turn in claudeflow. TOON is a drop-in serialization swap (no new service, no new secret) that cuts both, wherever the data is actually uniform enough to benefit.

---

## Section 39 — No em dash (—) in public-facing text (CEO directive 2026-08-03)

**Hard rule.** Never use the em dash (—) to join clauses in any text that ships externally: social posts (FB/LINE/IG captions), web copy, ad copy, LINE OA messages to users, any string a real customer or the public will read. It's an AI writing tell, not how Thai people actually write — Thai sentence flow uses conjunctions (แต่ / ซึ่ง / โดย / เพราะ), a new sentence, or just a space — not a dash splicing two clauses together.

- **Banned:** em dash (—) as a clause-joiner in public/outbound copy — captions, landing pages, ad creative, LINE messages sent to users, any "voice of the brand" text.
- **Not covered by this rule:** internal docs — wiki pages, ADRs, plans, this rules file itself — where em dash is just a formatting convention, not brand voice. The trigger for this rule was an internal analysis doc, but the rule targets public output, not internal notes.
- Applies across all brands (MoonieX, WarpClip, LungNote, Chatudo) and all content skills (`mooniex-content-skill`, FB caption pipeline, ad copy).
- When editing/reviewing copy before it ships, scan for — and rewrite as a real Thai sentence break.

> CEO (verbatim, 2026-08-03), pointing at a JTBD sentence CMO wrote: "ตัวนี้ มีแอทบจทั้งหมด ใน ประโยคเลย มันดูไม่เหมือน มนุษย์ พิม เลย ... ห้ามใช้ ในข้อความที่ต้อง Post หรือ เขียนในเว็ป หรือ ข้อความที่จะต้องส่งออกใน Public เด็ดขาด มันดูไม่เป็นมนุษย์ แล้วคนไทยไม่นิยมใช้กันในการ คั้น" — then repeated "—" alone several times to point at the exact character.

**Why:** em-dash clause-splicing is a recognizable LLM-generated-text pattern; real Thai copy doesn't write that way. Shipping it in public content makes brand voice read as obviously AI-written, undercutting the "talk like a normal person" register already required by §37.

---

## Section 40 — Tag parked LungNote todos with `[SID:<full-id>]` (CEO directive 2026-08-03, corrected 2026-08-03 after DEV verification)

**Hard rule.** When any session (CTO/CXO/DEV) adds a LungNote todo via
`add_todo` for work that originated in — or is being parked from — that
session, prefix the `text` with `[SID:<full-id>]`, where `<full-id>` is the
**complete, type-prefixed** id already used for spawn/log/task naming —
`cto-0b5934ec`, `cxo-...`, or `task-3c92a4c3`. **Keep the type prefix, do not
strip to bare 8-hex** — task ids and CTO/CXO spawn ids are different
namespaces, and a bare hex fragment can't tell you which one it is.

- Format: `[SID:cto-0b5934ec] Restart Claude Code session เพื่อเปิดใช้ 3 MCP connection ใหม่`
- Format (DEV task): `[SID:task-3c92a4c3] TEST — DEV-context check of IRON §40 tag convention`
- No LungNote schema change — this uses the existing `text` field, not a new
  column. (LungNote deliberately avoids schema changes against shared prod;
  see the `done`-bool-only precedent for todo state.)
- **Retrieval: use `list_todos` (with `include_done`/a large `limit`), not
  `search_notes`.** `search_notes` only queries `lungnote_notes` (title/body)
  — it does **not** search todo text, so it will not find a `[SID:...]` tag.
  Also note `list_todos` sorts by `due_at` with nulls last, so an undated
  tagged todo can fall off the default page — pass an explicit `limit` and
  `include_done:true` when hunting for a specific tag.
- The tag does **not** auto-load that session's context — pair with
  `/session-merge <full-id>` (CTO/CXO) or the session's log file for that.
  DEV task ids aren't chat sessions and have no `/session-merge` target;
  the tag there is provenance only ("who created this"), not a resume handle.
- Start using immediately — no code change required, just how every session
  calls `add_todo` from now on.

> CEO (verbatim, 2026-08-03): "เอาแบบ 1 แต่ใส่ SID:...." — confirming the
> `[SID:xxxxxxxx]` text-prefix format (not a separate `note_title`-per-session
> bucket), to start "เริ่มใช้เลยตั้งแต่ตอนนี้."

**Why:** todos parked mid-session are easy to lose track of which
conversation/context spawned them. A greppable session tag costs nothing
(no schema/dev work) and lets a future session — human or AI — trace a todo
back to its origin log instead of guessing.

**Correction (2026-08-03, verified via task-3c92a4c3):** the rule as
originally written was tested live via a DEV task and two things were wrong:
(1) it said `search_notes` does the lookup — it doesn't, `list_todos` does;
(2) it exemplified stripping the type prefix to bare 8-hex, which collides
DEV task ids with CTO/CXO session ids into one ambiguous namespace. Both
fixed above. **Known open gap:** the `lungnote` MCP server is not currently
wired into DEV worktree sessions — a DEV cannot call `add_todo` through the
sanctioned wrapper today (verified: absent from the DEV tool schema in
task-3c92a4c3, worked around via raw MCP stdio for that one verification).
Tracked as a follow-up infra task; until fixed, DEVs cannot practically
follow this rule through normal means.
