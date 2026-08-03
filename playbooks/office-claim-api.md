# Office Simulator — claim / heartbeat / release API

> Every agent that touches a `mooniex-webapp (Desk-*)` working copy
> MUST claim the desk first, beat the heartbeat while working, and
> release on stop. Per [IRON-RULES §15](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md#section-15--agent-role-memory--office-simulator-coordination) (`mooniex:IRON-RULES.md`).

The PreToolUse / UserPromptSubmit / Stop hooks at
`~/.claude/hooks/office-{pretool,userprompt,stop}.mjs` do this
automatically — but they only fire for cwds that match
`mooniex-webapp (Desk-*)`. Anything else (working in `LLMs/` as CTO,
running curl smoke tests, ad-hoc scripts) requires a manual call.

---

## Quickstart — first session at a desk

Run this checklist on every new Claude session BEFORE the first
`Write` / `Edit` / `Bash` that mutates anything.

```bash
# Step 1 — confirm cwd matches your desk folder
basename "$PWD"               # expect: mooniex-webapp (Desk-X)

# Step 2 — read your assigned role
cat ~/.claude/projects/*/memory/reference_office_role.md | head -20

# Step 3 — pull-before-edit (IRON §1)
git fetch origin
git pull --ff-only

# Step 4 — verify the office API token + base
cat ~/.claude/office.json     # api_base + token must both be set
```

If steps 1-4 pass, the **UserPromptSubmit hook auto-claims for you** on
the first prompt of the session — and re-claims at the start of every
turn. The Stop hook releases the desk at the end of every turn so
`/admin/office-simulator` shows real-time idle/working state.

For **non-`mooniex-webapp (Desk-*)` cwds** (CTO sitting in `/Users/gob/projects/LLMs/`, CMO scripts, etc.) the hook also auto-claims by reading the role memory file at `~/.claude/projects/<sanitized-cwd>/memory/reference_office_role.md` and parsing the `## Desk` line. No manual claim required for normal chat sessions.

Manual claim is still needed for:
- Pure shell scripts running outside any Claude session
- Smoke tests / curl probes
- Sessions where the role memory file is missing / unreadable

See [§ Manual claim from a non-Desk cwd](#manual-claim-from-a-non-desk-cwd) for those cases.

---

## Pick your desk by role

Every Claude session has a role assigned by the CEO and stored at
`~/.claude/projects/<sanitized-cwd>/memory/reference_office_role.md`.
Match the role to a desk **before** claiming.

| Role | Desk | Folder | Scope hint |
|------|------|--------|------------|
| Senior Developer (slot 1) | `Desk-A` | `mooniex-webapp (Desk-A)/` | `senior-dev-slot-1` |
| Senior Developer (slot 2) | `Desk-B` | `mooniex-webapp (Desk-B)/` | `senior-dev-slot-2` |
| Senior Developer (slot 3) | `Desk-C` | `mooniex-webapp (Desk-C)/` | `senior-dev-slot-3` |
| (TBD — pending CEO assignment) | `Desk-D` | `mooniex-webapp (Desk-D)/` | (set by CEO) |
| (TBD — pending CEO assignment) | `Desk-E` | `mooniex-webapp (Desk-E)/` | (set by CEO) |
| (TBD — pending CEO assignment) | `Desk-F` | `mooniex-webapp (Desk-F)/` | (set by CEO) |
| CTO | `Desk-CTO` | (no folder — cross-desk) | `cto-tech-leadership` |
| CMO | `Desk-CMO` | `mooniex-webapp (Desk-CMO)/` | `cmo` |

**Rules for picking a desk:**

1. **Desk-A / B / C are Senior Developer slots 1 / 2 / 3 ONLY.** No other role may sit there.
2. If the role memory says "Senior Developer (slot 1)" → claim **Desk-A**, never Desk-B / Desk-C.
3. If the role memory says CTO → claim **Desk-CTO** manually (no folder; CTO works cross-desk via API).
4. Two Senior Devs cannot sit at the same slot. If your role says slot 2 but `Desk-B` returns 409 → escalate to CEO; do NOT switch slots silently.
5. Desk-D / E / F roles are not yet set — do not claim until CEO assigns.
6. Roles are admin-assigned via `/admin/office-simulator` — never self-promote (§15 #1).

---

## Endpoints

```
POST https://www.mooniex.com/api/office/claim
POST https://www.mooniex.com/api/office/heartbeat
POST https://www.mooniex.com/api/office/release
GET  https://www.mooniex.com/api/office/state
```

All three writes need `Authorization: Bearer ${OFFICE_TOKEN}`. The
token lives at `~/.claude/office.json`. `GET /state` is public so
diagnostics work without it.

### Folder-usage rules (per-desk)

- **Edit only inside your own desk folder.** Do NOT touch
  `/Users/gob/projects/mooniex-webapp (Desk-X)/` if your desk is Desk-Y.
  Cross-desk audits are CTO-only (§15 + memory file).
- **Pull before edit, push after commit, every time** (§1).
- **Hot Zones** (`AGENTS.md`, `IRON-RULES.md`, `INDEX.md`,
  `CROSS-FOLDER-MAP.md`, hot files in §10) → serialize. Only one
  desk edits at a time. Coordinate via the office state API.
- **Migrations are timestamped** — collisions add domain prefix (§4).
- **Wiki sync goes first** — push `LLMs/*` BEFORE pushing project
  changes that depend on the rule (LLMs/CLAUDE.md "Update flow").

### API rules (per-desk)

- **`OFFICE_TOKEN`** lives in `~/.claude/office.json` and Vercel env.
  Never paste it in chat / commits / logs.
- **One claim at a time per session.** Don't claim two desks from
  the same Claude session — that's a §15 #2 violation.
- **Heartbeat every ≥60s** while working. The hook does this; manual
  sessions do it themselves. Stale 20m → auto-release.
- **Release on stop.** `reason=work_done` for normal completion,
  `reason=stop_hook` for compact / clear / session end.
- **`agent_id` is deterministic** — same host + cwd + UTC day → same
  id. Re-claims after a hook miss don't look like a new agent.

---

## Lifecycle (per-turn — 2026-04-27 update)

The desk lifecycle now follows assistant **turn boundaries**, not session boundaries. Hook chain per turn:

```
turn N start              ...turn body...               turn N end
─────────────             ─────────────────             ─────────────
UserPromptSubmit          PreToolUse (per tool)          Stop
   │                          │                            │
   ▼                          ▼                            ▼
claim-or-heartbeat        heartbeat (throttled 60s)    release(turn_end)
   │                          │                            │
   ▼                          ▼                            ▼
desk = working             stays working               desk = idle
```

State diagram:

```
┌──────────┐  UserPromptSubmit   ┌──────────┐
│   idle   │ ──────claim───────▶ │ working  │
│ (no row) │                     │ (claimed)│
└──────────┘                     └──────────┘
     ▲                                │
     │                                │ Stop hook
     │   release (turn_end)           │
     └────────────────────────────────┘

         OR — backstop paths

         ──── Stop never fired ──────▶ stale_sweep cron auto-releases @ 20min
         ──── /clear / compact ──────▶ Stop hook still fires → release
```

Implications:
- `/admin/office-simulator` shows real-time per-turn working/idle state
- Two agents can rotate on Desk-A across turns (agent A turn 1, agent B turn 2 — both legit)
- 20-min stale sweep only catches crashed sessions where Stop didn't fire

## 1. Claim (before any edit)

```bash
TOK=$(node -e 'process.stdout.write(JSON.parse(require("fs").readFileSync(process.env.HOME+"/.claude/office.json","utf8")).token)')

# Compute deterministic agent_id (matches the PreToolUse hook formula)
HOST=$(hostname)
DAY=$(date -u +%Y-%m-%d)
SEED="${HOST}|${PWD}|${DAY}"
AID="claude-$(printf '%s' "$SEED" | shasum -a 256 | awk '{print $1}' | cut -c1-12)"

curl -X POST https://www.mooniex.com/api/office/claim \
  -H "Authorization: Bearer $TOK" \
  -H "Content-Type: application/json" \
  -d "{
    \"desk_id\":       \"Desk-A\",
    \"agent_id\":      \"$AID\",
    \"agent_kind\":    \"claude\",
    \"agent_label\":   \"Claude · $(hostname) · $DAY · SrDev-1\",
    \"agent_role\":    \"Senior Developer\",
    \"branch\":        \"$(git rev-parse --abbrev-ref HEAD 2>/dev/null)\",
    \"task_summary\":  \"<one-line description of work\"
  }"
```

Response on success — keep `claim_id`, you need it for heartbeat + release:

```json
{
  "ok": true,
  "claim_id": "dab749fb-def2-4876-a2ae-8492b6e45260",
  "desk":     { "id": "Desk-A", "folder_path": "...", "display_x": 80, "display_y": 120 },
  "role":     null
}
```

> `role` in the response comes from `webapp_office_agents` joined
> on `agent_id`. It stays `null` until the admin assigns a row via
> `/admin/office-simulator` → "Assign role". Until then the
> `agent_role` you pass in the claim payload is hint-only — the
> sprite label uses it but the response field is the DB truth.

Errors:
- `409 occupied` — another agent owns this desk; pick another or wait
- `401 unauthorized` — `OFFICE_TOKEN` missing / wrong
- `400 invalid_payload` — desk_id not in the catalog or agent_kind bad

## 2. Heartbeat (every ~60s while editing)

```bash
curl -X POST https://www.mooniex.com/api/office/heartbeat \
  -H "Authorization: Bearer $TOK" \
  -H "Content-Type: application/json" \
  -d "{\"claim_id\":\"$CLAIM_ID\"}"
```

Response: `{"ok":true}` — bumps `last_heartbeat_at`.
`404 stale_or_released` — claim is gone; re-claim before editing again.

The userprompt hook beats every prompt submission (≥60s throttle); no
need to call manually unless your session is fully autonomous.

## 3. Release (when work is done)

```bash
curl -X POST https://www.mooniex.com/api/office/release \
  -H "Authorization: Bearer $TOK" \
  -H "Content-Type: application/json" \
  -d "{\"claim_id\":\"$CLAIM_ID\",\"reason\":\"work_done\"}"
```

Response: `{"ok":true}`. Idempotent — releasing an already-released
claim is harmless.

`reason` is free text. Common values:
- `work_done` — normal completion
- `stop_hook` — Claude Code session ended (the Stop hook uses this)
- `force` — admin force-released via `/admin/office-simulator`
- `stale_sweep` — cron noticed the heartbeat went silent
- `role_assignment` — releasing to re-claim with a fresh role label

## 4. Inspect state

```bash
curl -s https://www.mooniex.com/api/office/state | jq '.active'
```

Returns the full snapshot used by `/admin/office-simulator`. Public read.

```bash
# pretty per-desk view
curl -s https://www.mooniex.com/api/office/state | python3 -c "
import sys, json
d = json.load(sys.stdin)
for k, v in d['active'].items():
    if v:
        print(f'{k:10} CLAIMED agent={v[\"agent_id\"][:20]} role={v.get(\"agent_role\")} label={v.get(\"agent_label\")}')
    else:
        print(f'{k:10} vacant')
"
```

## Common patterns

### Determinist agent_id

Hooks compute it as:

```js
const day = new Date().toISOString().slice(0, 10);   // UTC date, NOT local
const seed = `${os.hostname()}|${process.cwd()}|${day}`;
const agent_id = "claude-" + crypto.createHash("sha256").update(seed).digest("hex").slice(0, 12);
```

Same host + cwd + UTC day = same id. Re-claims after a hook miss
don't look like a new agent. Note: UTC day rolls over at 07:00 BKK,
so `agent_id` changes mid-morning Bangkok time — that's expected.

### Manual claim from a non-Desk cwd

CTO working in `/Users/gob/projects/LLMs/` (the LLMs wiki) — hooks
don't fire there. Manual flow:

```bash
TOK=$(node -e 'process.stdout.write(JSON.parse(require("fs").readFileSync(process.env.HOME+"/.claude/office.json","utf8")).token)')

CLAIM=$(curl -s -X POST https://www.mooniex.com/api/office/claim \
  -H "Authorization: Bearer $TOK" \
  -H "Content-Type: application/json" \
  -d '{
    "desk_id":      "Desk-CTO",
    "agent_id":     "claude-cto-manual",
    "agent_kind":   "claude",
    "agent_label":  "CTO wiki edits",
    "agent_role":   "CTO",
    "branch":       "main",
    "task_summary": "Update IRON-RULES + new playbook"
  }')

CID=$(echo "$CLAIM" | jq -r .claim_id)

# … do the work, push commits …

curl -X POST https://www.mooniex.com/api/office/release \
  -H "Authorization: Bearer $TOK" \
  -H "Content-Type: application/json" \
  -d "{\"claim_id\":\"$CID\",\"reason\":\"work_done\"}"
```

### Re-claim with updated role label

If your role changed mid-session and you want the sprite label to
reflect it, release + re-claim with the new `agent_role`:

```bash
curl -X POST https://www.mooniex.com/api/office/release \
  -H "Authorization: Bearer $TOK" -H "Content-Type: application/json" \
  -d "{\"claim_id\":\"$OLD_CID\",\"reason\":\"role_assignment\"}"

# then re-claim with the new agent_role + agent_label …
```

The DB `agent_role` column on `webapp_office_agents` is admin-only —
your `agent_role` in the claim payload is a label hint, not a DB
write. The CEO PATCHes the canonical role via
`PATCH /api/admin/office/agent/[agent_id]`.

### Mid-task detection of a stale heartbeat

If heartbeat returns `404 stale_or_released`, the claim has been
swept. Re-claim before any further edits — otherwise IRON-RULES §1
"one agent per desk" can be silently violated.

---

## What NOT to do

- ❌ Edit a desk file without an active claim — even read-only inspection
  is fine, but the moment you write, you owe a claim.
- ❌ Sit at a desk that doesn't match your role memory. Slot-2 Senior
  Dev claiming Desk-A is a §15 #2 violation.
- ❌ Use someone else's `claim_id` to heartbeat / release — the desk is
  not yours.
- ❌ Hard-code `OFFICE_TOKEN` in source / commits / chat. It belongs
  in `~/.claude/office.json` (mode 0600) and Vercel env only.
- ❌ Skip release on `Stop` — desks accumulate stale claims and the
  20-min sweep is a cushion, not a substitute.
- ❌ Self-rebrand by changing `agent_role` in the claim payload to a
  role you weren't assigned. The role memory is the source the CEO
  approved; the claim payload mirrors it, doesn't override.
