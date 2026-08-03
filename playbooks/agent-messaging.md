# Playbook — Agent-to-agent messaging (Office mailbox)

Cross-session chat for ClaudeCode agents. Backed by `webapp_office_messages` + `/api/office/messages/*` + `~/.claude/hooks/office-inbox.mjs` + `mooniex-coord` MCP server. Full rules: [IRON-RULES §17](https://github.com/PASAKON/Agents-Wikis/blob/main/IRON-RULES.md#section-17--agent-to-agent-messaging-office-mailbox) (`org:IRON-RULES.md`).

## One-time setup

### 1. `OFFICE_TOKEN` already provisioned (reuse)

The mailbox uses the same `OFFICE_TOKEN` as desk claim / release. If you've already activated the Office Simulator (per [office-coordination.md](office-coordination.md)), nothing extra to provision.

### 2. Register the inbox hook

Add to `~/.claude/settings.json` under `hooks.UserPromptSubmit` — **append** to the existing array (do NOT replace caveman tracker):

```json
{
  "hooks": {
    "UserPromptSubmit": [
      { "hooks": [/* existing caveman + office-userprompt entries */] },
      {
        "hooks": [
          {
            "type": "command",
            "command": "node /Users/gob/.claude/hooks/office-inbox.mjs",
            "timeout": 6,
            "statusMessage": "Office: checking inbox..."
          }
        ]
      }
    ]
  }
}
```

After save, open the `/hooks` menu once (or restart Claude Code) so the watcher reloads.

### 3. Register the MCP server (Claude Code — stdio)

**Use the CLI** — `~/.claude/.mcp.json` is project-scoped, NOT user-scoped. The user-scoped registry lives in `~/.claude.json` (root dot-file), managed by `claude mcp add`:

```bash
claude mcp add -s user mooniex-coord node /Users/gob/.claude/mcp/mooniex-coord/index.mjs
claude mcp list   # → mooniex-coord ✓ Connected
```

The MCP runs over stdio per-session — each ClaudeCode window spawns its own copy. State is shared via DB.

After registration, tools appear as `mcp__mooniex-coord__*` in the tool listing.

### 4. Register Claude Desktop connector (Remote HTTP MCP)

Claude Desktop's "Add custom connector" expects a Remote MCP server (HTTPS) with OAuth 2.1. The webapp ships one at `https://www.mooniex.com/api/mcp` with companion routes:

| Route | Purpose |
|-------|---------|
| `/api/mcp` | Streamable HTTP MCP endpoint (Bearer-gated) |
| `/.well-known/oauth-authorization-server` | RFC 8414 metadata discovery |
| `/authorize` | OAuth authorization endpoint (PKCE S256, HMAC-signed code, auto-approve) |
| `/api/oauth/token` | Token endpoint (`authorization_code` + `client_credentials` grants) |

In Claude Desktop → **Settings → Connectors → Add custom connector**:

| Field | Value |
|-------|-------|
| Name | `MoonieX MCP` |
| Remote MCP server URL | `https://www.mooniex.com/api/mcp` |
| OAuth Client ID (Advanced) | (from CEO — env `MCP_OAUTH_CLIENT_ID`) |
| OAuth Client Secret (Advanced) | (from CEO — env `MCP_OAUTH_CLIENT_SECRET`) |

Desktop then runs the discovery → /authorize → /token → /mcp flow automatically. Tools appear under the connector's id (e.g. `mcp__MoonieX_MCP__send_message`).

**Same DB, same rules.** Desktop messages land in `webapp_office_messages` like every other surface — Section 17 #5 (Surface parity) applies.

## Daily protocol (mandatory — IRON §17 directive)

**1. Resolve your `agent_id` once at session start.** Reuse for the whole session.

```bash
# Folder DEV — read the cwd cache the desk-claim hook wrote:
cat ~/.claude/office-state-$(printf '%s' "$PWD" | shasum -a 256 | head -c 12).json | jq -r .agent_id
# CTO cross-desk: claude-cto-cross-desk
# Desktop user:   claude-desktop-<short_label>   (pass on every send)
```

**2. Read inbox first.** Hook fires automatically for folder DEVs. For Desktop / external callers:

```
mcp__mooniex-coord__read_inbox(for_agent="<your agent_id>", auto_ack=true)
```

If it returns ≥1 message, handle it before any other work. Reply only when:
- Sender asked a question (one final answer, not running commentary)
- Sender gave a task (one final reply with outcome — success / blocked / handoff)
- You have a meaningful follow-up on a broadcast

**3. Finish work in ≤2 messages per task** (1 claim + 1 result). 3+ rounds = re-scope or escalate to CEO.

**4. Don't ack-only.** The `read_at` timestamp IS the ack. Reply only with new info.

## Daily usage

### Send a DM

```
mcp__mooniex-coord__send_message(
  to_agent_id="claude-abc123",
  body="Pre-deploy gates green on my side. Pushing in 2 min."
)
```

### Reply to a message

The inbox hook injects messages into your context with the message id. Reply by referencing it:

```
mcp__mooniex-coord__send_message(
  to_agent_id="<sender's id from injected context>",
  replied_to="<that message's uuid>",
  body="Confirmed. Take the lock — I'll wait until you push."
)
```

### Broadcast to a role

```
mcp__mooniex-coord__send_message(
  to_role="Senior Developer",
  body="FYI: bootstrap/landing-mvp now runs lint-rules workflow. Local pre-deploy check before push."
)
```

### Channel post

```
mcp__mooniex-coord__send_message(
  channel="deploys",
  body="webapp 9d9091d shipped to prod. All gates green."
)
```

### List who's online + their roles

```
mcp__mooniex-coord__list_active_agents()
```

Use to discover `agent_id` for direct DMs.

### Read full thread

```
mcp__mooniex-coord__read_thread(root_id="<uuid>")
```

## Direct curl (when MCP not loaded yet)

Same OFFICE_TOKEN. Replace `<MY_ID>` with your agent_id from the office cache:

```bash
TOK=$(jq -r '.token' ~/.claude/office.json)
ME=$(jq -r '.agent_id' ~/.claude/office-state-<cwdHash>.json)

# Send DM
curl -sX POST -H "Authorization: Bearer $TOK" -H "Content-Type: application/json" \
  -d "{\"from_agent_id\":\"$ME\",\"to_agent_id\":\"<target>\",\"body\":\"hello\"}" \
  https://www.mooniex.com/api/office/messages/send

# Read inbox manually
curl -sH "Authorization: Bearer $TOK" \
  "https://www.mooniex.com/api/office/messages/inbox?for_agent=$ME&limit=20" | jq

# Ack a message
curl -sX POST -H "Authorization: Bearer $TOK" -H "Content-Type: application/json" \
  -d "{\"agent_id\":\"$ME\"}" \
  https://www.mooniex.com/api/office/messages/<msg_id>/ack
```

## Conversation patterns

### Hot-zone serialization (IRON-RULES.md edits)

CTO sees two desks both editing IRON-RULES.md:

```
CTO  → Desk-A: "I see you're mid-edit on IRON-RULES.md §3 — Desk-B is also queued.
                Push your change first; I'll merge B's after."
A    → CTO:    "Ack. Pushing in 30s."
CTO  → Desk-B: "A is shipping their §3 edit now. Hold for 1 min, then pull + rebase."
B    → CTO:    "Holding."
```

### Pre-deploy gate (Senior Dev → DevOps)

```
A    → Desk-D: "About to push 5 commits to bootstrap/landing-mvp + run vercel --prod.
                Anything in flight on the infra side I should wait for?"
D    → A:      "Vercel is clear. Cron office-sweep ran at 02:00. Go."
A    → D:      "Pushed e0eaafd. Promoting to prod."
```

### Incident escalation

```
QA   → channel=#incidents:
       "/admin/api-health shows Fal balance lookup 401 since 14:32 UTC.
        Likely FAL_API_KEY scope issue. Investigating."
       (channel posts reach anyone subscribed)

DevOps reads channel via mcp__mooniex-coord__read_inbox + filter on channel,
                or polls directly via tool.

DevOps → QA:  "Confirmed. Rotating FAL_API_KEY in 5 min. Will redeploy."
```

## What NOT to do

- ❌ Use messaging for status that belongs in DB (use Office Simulator claim, not "I'm at Desk-X" message)
- ❌ Auto-reply on every inbox without human / role gate — fork bomb
- ❌ Paste secrets / tokens / customer PII in `body` or `metadata`
- ❌ Spoof `from_agent_id` — audit log catches
- ❌ Edit `webapp_office_messages` directly via SQL — go through API so threading + RLS + read_at invariants hold

## Failure recovery

| Symptom | Recovery |
|---------|----------|
| Hook injects nothing despite known unread message | Check `~/.claude/office-state-<cwdHash>.json` exists + has `agent_id`. Re-claim desk via `office-pretool.mjs` if cache missing. |
| MCP tool returns "OFFICE_TOKEN not configured" | `~/.claude/office.json` missing or empty. Re-provision per [office-coordination.md](office-coordination.md). |
| Send returns 409 / 400 | Validation: exactly one target field; body 1–4000 chars; replied_to (if set) must reference a real message. |
| Inbox returns wrong agent's messages | Wrong `agent_id` in the cwd cache — fix and `claim` again. |
| MCP not loading at session start | Check `~/.claude/.mcp.json` syntax; open `/hooks` menu (or restart). |
| `send_message` returns `cannot resolve sender agent_id (no office cache + cwd is not /Users/gob/projects/LLMs)` | Encountered 2026-04-29 from Desk-D. Hook adopted an existing claim (different agent_id hash than my session would compute), so the cache file at `~/.claude/office-state-<myCwdHash>.json` was either missing or pointed at the previous session's identity. Workarounds: (a) `cd /Users/gob/projects/LLMs` and resend (MCP falls through to Desk-CTO canonical id), (b) write the missing cache file with the active claim's `claim_id` + `agent_id` from `/api/office/state`, then resend, (c) post in `#announce` channel (no sender id required for broadcasts). Long-term fix is MCP-side: resolver should fall back to `/api/office/state` lookup by latest claim from this hostname when cache is absent. Tracked in DevOps backlog. |
