# Playbook — Office Simulator + desk coordination

Live coordination of which agent sits at which webapp desk. Backed by:
- Tables: `webapp_office_desks` + `webapp_office_claims` + `webapp_office_agents`
- API: `/api/office/{state,claim,heartbeat,release}` + `/api/cron/office-sweep`
- UI: `/admin/office-simulator`
- Hooks: `~/.claude/hooks/office-{pretool,userprompt,stop}.mjs`

Full rules: [IRON-RULES §1 desk model](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md#section-1--desk-model-one-agent-per-folder) (`mooniex:IRON-RULES.md`) + [§15 agent role + coordination](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md#section-15--agent-role-memory--office-simulator-coordination) (`mooniex:IRON-RULES.md`).

## One-time setup

### 1. Generate + provision `OFFICE_TOKEN`

```bash
TOKEN=$(openssl rand -hex 32)

# Vercel
printf "%s" "$TOKEN" | vercel env add OFFICE_TOKEN production
printf "%s" "$TOKEN" | vercel env add OFFICE_TOKEN preview

# Local hook config
cat > ~/.claude/office.json <<EOF
{
  "api_base": "https://www.mooniex.com",
  "token": "$TOKEN"
}
EOF
chmod 600 ~/.claude/office.json
```

### 2. Apply migration (auto via db-migrate Action on push)

The migration `202604290100__webapp_office_simulator.sql` ships with the next push. The Action applies it and the 3 default desks seed automatically. **Note**: the migration originally seeded `Desk-1`; a follow-up migration `202604290200__webapp_office_desk_1_to_a.sql` renames that row to `Desk-A` (see `MIGRATIONS.md` 2026-04-27 entry). Effective desk ids today: `Desk-A`, `Desk-B`, `Desk-C`.

### 3. Register Claude Code hooks

Add to `~/.claude/settings.json` under `hooks` (merge with existing):

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Write|Edit|Bash",
      "hooks": [{
        "type": "command",
        "command": "node /Users/gob/.claude/hooks/office-pretool.mjs",
        "timeout": 8
      }]
    }],
    "UserPromptSubmit": [{
      "hooks": [{
        "type": "command",
        "command": "node /Users/gob/.claude/hooks/office-userprompt.mjs",
        "timeout": 5
      }]
    }],
    "Stop": [{
      "hooks": [{
        "type": "command",
        "command": "node /Users/gob/.claude/hooks/office-stop.mjs",
        "timeout": 5
      }]
    }]
  }
}
```

After save, open the Claude Code `/hooks` menu once (or restart) so the watcher reloads.

### 4. Open the simulator

`https://www.mooniex.com/admin/office-simulator`

### 5. Assign roles

Click each occupied desk → drawer → "Assign / edit role" → fill:
- Display name (e.g. "Claude Opus 4.7 · Desk-A")
- Role (e.g. "CTO" / "Senior Developer" / "ML Engineer" / "DevOps Engineer" / "QA Engineer")
- Role brief — the 1-2 sentence the agent quotes when asked
- Responsibilities — one per line
- Scope tags — comma-separated

Audit row written to `webapp_audit_log` on every save.

## Daily flow (agent perspective)

1. Open Claude Code in `/Users/gob/projects/mooniex-webapp (Desk-X)/`
2. Start working — first `Write` / `Edit` / `Bash` triggers the PreToolUse hook → POST `/api/office/claim`
3. If success: silently proceed
4. If 409 occupied: hook blocks the tool with a `permissionDecision: deny` and an explanation. Agent reads the message, switches folder, retries
5. Subsequent edits heartbeat (≤1/min)
6. On `/clear`, `/resume`, compact, or normal exit: Stop hook releases the desk

## "What's your role?" — agent answer pattern

> "ผมรับหน้าที่ในตำแหน่ง **\[role from claim\]** — \[role_brief from memory\].
>
> ความรับผิดชอบของผม:
> - <responsibility 1>
> - <responsibility 2>
> - …
>
> ขอบเขต: <scope tags>"

If unassigned:
> "ยังไม่ได้รับมอบหมายตำแหน่ง — admin ยังไม่ assign ใน `/admin/office-simulator`."

## Force-release a stuck desk

UI: `/admin/office-simulator` → click desk → drawer → "Force release". Writes a release with `reason=force` and audits.

API:
```bash
curl -X POST https://www.mooniex.com/api/office/release \
  -H "Authorization: Bearer $OFFICE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"claim_id":"<uuid>","reason":"force"}'
```

## Failure modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Hook silent on first edit | `~/.claude/office.json` missing or token wrong | Re-run setup step 1 |
| `permissionDecision: deny` blocking work | Desk occupied | Switch to `Desk-B` / `Desk-C`, or force-release if you know the other agent crashed |
| Cron sweep not running | `OFFICE_TOKEN` not set in Vercel preview env | Add via `vercel env add` |
| Heartbeats failing 404 | Cached `claim_id` was released by sweep | Hook auto-re-claims on next call |
| Realtime not updating UI | RLS / publication missing | Migration adds the publication; verify with `select * from pg_publication_tables where tablename='webapp_office_claims'` |

## What NOT to do

- ❌ Edit `webapp_office_agents` directly to give yourself a role. Use the admin UI.
- ❌ Read another agent's `~/.claude/projects/<other>/memory/` directory.
- ❌ Skip the hook by exporting `OFFICE_TOKEN=""`. Coordination breaks silently.
- ❌ Hardcode desk ids in business code — read from `webapp_office_desks`.
