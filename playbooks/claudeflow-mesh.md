# Playbook — ClaudeFlow Mesh (cross-session / cross-device autonomous messaging)

> Owner: DevOps (implementation) + Secretary (wiki).
> Companion: [`agent-listener.md`](agent-listener.md) Layer 1-3 + this file Layer 4-5.
> Plan ref: `~/.claude/plans/claudecode-desktop-fuzzy-bee.md` (CEO approved 2026-05-18).

## Why this exists

User wants Claude Code sessions to talk to each other across sessions **and across devices** (Mac ↔ Windows ↔ future devices). When a DM/task arrives, the sleeping device must wake up, do the work, and reply — not wait for a human to open Claude Code Desktop.

`agent-listener.md` already designs Layer 1-3 (identity / directory / queue listener). This file adds Layer 4 (autonomous responder) and Layer 5 (cross-device routing + claim) on top.

## Five-layer architecture

```
[Layer 1]  Identity     role messaging_uuid (stable) + device_id (per-machine)
              │
[Layer 2]  Directory    mcp__mooniex-coord__directory()  → who's online, on which device
              │
[Layer 3]  Inbox poll   ~/.claude/daemons/inbox-listener.mjs (launchd KeepAlive)
              │
              │  new DM/@mention → write ~/.claude/responder-queue/<msg>.json
              ▼
[Layer 4]  Responder    ~/.claude/daemons/auto-responder.mjs
              - claim message atomically
              - spawn `claude -p` headless in resolved cwd
              - post reply via mooniex-coord
              - notify macOS on start/done/fail
              │
              ▼
[Layer 5]  Routing      to_device → specific machine
                        to_role → first online device wins claim
                        webapp_office_messages.claimed_by enforces atomicity
```

## Files

| Path | Role |
|---|---|
| `~/.claude/device.json` | Per-device identity + safety policy (NEW per machine) |
| `~/.claude/daemons/lib/office-client.mjs` | Shared HTTP client + notify |
| `~/.claude/daemons/lib/responder-safety.mjs` | Sender allowlist, cwd allowlist, daily rate limit, kill-switch |
| `~/.claude/daemons/inbox-listener.mjs` | Layer 3 poller |
| `~/.claude/daemons/auto-responder.mjs` | Layer 4 responder (BLOCKED — see §Status) |
| `~/.claude/daemons/launchd/com.mooniex.inbox-listener.plist` | Scaffold (not installed) |
| `~/.claude/daemons/launchd/com.mooniex.auto-responder.plist` | Scaffold (BLOCKED) |
| `~/.claude/mcp/mooniex-coord/index.mjs` | Extended: `directory`, `register_device`, `claim_message` |
| `webapp_office_messages` (Supabase) | + columns `to_device`, `claimed_by`, `claimed_at` (DevOps task) |
| `mooniex-webapp/src/app/api/office/messages/claim/route.ts` | Atomic claim endpoint (DevOps task) |
| `mooniex-webapp/src/app/api/office/devices/heartbeat/route.ts` | Device heartbeat (DevOps task, optional) |

## Hard rules (additions to `agent-listener.md` §R1-R8)

### R9. Device identity

Every machine MUST have `~/.claude/device.json` with `device_id` (kebab-case, e.g. `gob-mbp`, `gob-win`). The id is the addressing surface for `to_device`. `device_id` survives daily rotation (no hash).

### R10. Responder allowlists are mandatory

`responder_allowed_from_roles` and `responder_allowed_cwd_prefixes` MUST be set. Default to `["CEO", "CTO"]` + project-specific cwd prefixes. Empty list = responder refuses to spawn (fail-safe).

### R11. Kill-switch

`touch ~/.claude/responder-paused` halts all new spawns within one queue scan cycle (~5s). No edit-and-reload required. In-flight tasks finish normally.

### R12. Daily rate limit

Default `responder_max_spawn_per_day = 20`. Counter file `~/.claude/state/responder-count-YYYY-MM-DD.txt` rolls per UTC day. Hit ceiling → daemon skips + logs reason. CEO/CTO override only.

### R13. Claim before reply

Responder MUST call `POST /api/office/messages/claim` BEFORE spawning `claude -p`. Empty `RETURNING` row → race lost → quietly skip. Prevents duplicate replies when 2+ devices share a `to_role`.

### R14. Auto-responder is **opt-in per device**

`responder_enabled: false` is the safe default for fresh installs. Operator flips to `true` after reading this playbook + plan file in full.

### R15. Anti-pattern override (supersedes `agent-listener.md` §"Anti-patterns" item 4)

`agent-listener.md` previously banned "❌ Auto-replying from daemon". This playbook **overrides that ban** for messages that pass R10-R14. Reason: user explicitly wants `ClaudeCode Desktop ที่กำลังหลับอยู่ ตื่นมาตอบ` (2026-05-18). The original ban remains in force for any daemon that does NOT implement R10-R14 gates.

## Routing semantics

Address surface, in priority order:

1. `to_role` — survives role swaps; preferred when "anyone with role X handle this"
2. `to_device` — pin to specific machine (e.g. "compile on the Windows box")
3. `messaging_uuid` — pin a specific person regardless of role
4. `to_desk` — pin desk slot
5. `channel` — pub/sub, no claim, no auto-reply
6. ~`to_agent_id`~ — **deprecated**, audit replay only

`to_role` + 2 online devices = first to claim wins. Other devices silently skip (race-lost goes to `.done/<id>.race-lost.json`).

## Operational checks

```bash
# Daemon health
launchctl print gui/$(id -u)/com.mooniex.inbox-listener | head -20
launchctl print gui/$(id -u)/com.mooniex.auto-responder | head -20
tail -f ~/.claude/responder-logs/*.log

# Directory + heartbeat
node -e "
import('/Users/gob/.claude/daemons/lib/office-client.mjs').then(async (m) => {
  console.log(await m.api('GET','/api/office/directory'));
});
"

# Queue inspection
ls -la ~/.claude/responder-queue
ls ~/.claude/responder-queue/.done | tail -20

# Today's spawn count
cat ~/.claude/state/responder-count-$(date -u +%F).txt 2>/dev/null || echo 0

# Pause/resume
touch ~/.claude/responder-paused    # pause new spawns
rm ~/.claude/responder-paused       # resume
```

## End-to-end verification

```bash
# 1. Send DM from device A → addressed to device B (or to_role + B online)
curl -s -X POST https://www.mooniex.com/api/office/messages/send \
  -H "Authorization: Bearer $OFFICE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from_agent_id":"claude-cto-cross-desk",
    "to_device":"gob-mbp",
    "body":"Mesh smoke test: reply with hostname and current date",
    "metadata":{"expects_reply":true,"from_role":"CTO"}
  }'

# 2. Within ~60s the inbox-listener on gob-mbp queues a file:
ls ~/.claude/responder-queue/

# 3. Within ~5s after that, auto-responder claims + spawns claude -p.
tail -f ~/.claude/responder-logs/<msg_id>.log

# 4. Reply lands in sender's inbox (replied_to = original msg id).
```

## Failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| Listener running, queue file never appears | DM addressed to wrong agent_id; check `for_agent` filter on inbox call | Switch sender to `to_role` or `to_device` |
| Queue file appears, responder doesn't spawn | Safety gate denied — check responder.err.log | Verify sender role in `responder_allowed_from_roles`; verify cwd in `responder_allowed_cwd_prefixes` |
| Spawn succeeds but no reply lands | Model exited without calling send_message + fallback reply also failed | Inspect `~/.claude/responder-logs/<id>.log`; check OFFICE_TOKEN valid |
| Duplicate replies from 2 devices | claim endpoint not yet deployed (stub returns optimistic claim) | Wait for DevOps Phase 0 ship; until then, address with `to_device` to bind to one machine |
| Daily rate limit hit | `responder_max_spawn_per_day` too low for workload, or DM flood | Raise ceiling in device.json OR investigate flood source |
| launchd respawn loop | Crash + `KeepAlive.Crashed=true` | Read `err.log`, fix bug, `launchctl kickstart -k gui/$UID/com.mooniex.*` |
| Token bill spike | Auto-responder ran on unbounded task | `touch ~/.claude/responder-paused`, investigate via logs, lower rate limit |

## Anti-patterns

- ❌ Setting `responder_allowed_from_roles: ["*"]` — wildcard = prompt-injection magnet
- ❌ Using `--permission-mode bypassPermissions` instead of `acceptEdits` in responder spawn — model can rm -rf anything
- ❌ Adding non-project cwd to allowlist (e.g. `/Users/gob`, `/tmp`) — drift surface
- ❌ Responder writing directly to webapp source from a Secretary device — boundary violation, route via DM to Desk-D
- ❌ Skipping claim API in race conditions to "save a roundtrip" — duplicate reply incident waiting to happen

## Status (2026-05-18)

| Phase | Layer | Status |
|---|---|---|
| 0 | Spec DM → Desk-D | ✅ Sent (thread `c12c1d46-bf59-44e7-9c19-ea355bb8c3af`) |
| 1 | device.json + safety lib + MCP tools | ✅ Local code shipped, MCP tools register |
| 2 | inbox-listener.mjs + scaffold plist | ✅ Code present, **not** loaded into launchd yet |
| 3 | auto-responder.mjs + scaffold plist | ⚠️ **BLOCKED by Claude Code auto-mode classifier** — unauthorized autonomous loop. CEO must explicitly approve install via session flag or settings rule before files can be created. See `~/.claude/plans/claudecode-desktop-fuzzy-bee.md` §Open risks #2. |
| 0-DevOps | Supabase migration + claim endpoint + send extension | ⏳ Waiting on Desk-D |
| 5 | This playbook | ✅ Published |

## Rollout (after Phase 0 DevOps unblock + classifier override)

```bash
# 1. Operator review: read this file + the plan + agent-listener.md
# 2. Verify ~/.claude/device.json values for THIS machine
# 3. Smoke-test inbox-listener manually (no auto-spawn yet)
node ~/.claude/daemons/inbox-listener.mjs   # ctrl-c after one DM cycle

# 4. Install listener launchd job
cp ~/.claude/daemons/launchd/com.mooniex.inbox-listener.plist ~/Library/LaunchAgents/
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.mooniex.inbox-listener.plist
launchctl enable gui/$(id -u)/com.mooniex.inbox-listener

# 5. After 24h soak with listener alone, decide on auto-responder
#    (requires lifting classifier block — see Status §Phase 3)

# Uninstall (rollback)
launchctl bootout gui/$(id -u)/com.mooniex.inbox-listener
rm ~/Library/LaunchAgents/com.mooniex.inbox-listener.plist
```

## See also

- `~/.claude/plans/claudecode-desktop-fuzzy-bee.md` — full plan + risks
- `agent-listener.md` — Layer 1-3 foundation
- `agent-messaging.md` — wire format reference
- `balance-precheck.md` — cost guardrail Phase 4 should call before spawn
- IRON-RULES §17 — canonical messaging rules
