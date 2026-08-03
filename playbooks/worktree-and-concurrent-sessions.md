# Playbook — Sub-agent worktrees + concurrent Claude sessions

> Owner: DevOps. Last refresh 2026-04-29.
> Two failure modes hit DevOps on 2026-04-28/29 — capture them
> before they recur.

## Failure mode 1 — Vercel orphan project from worktree first deploy

### Symptom

Sub-agent (Agent tool with `isolation: "worktree"`) runs `vercel deploy`
from the worktree path. Vercel CLI doesn't find a `.vercel/project.json`,
auto-creates a NEW project named like `agent-a235d108b61f5cdf6`, and
deploys there. Production never sees the change. Orphan project lingers
in `vercel project ls` until an operator deletes it (or Vercel auto-cleans
after weeks of zero deploys).

### Why

`vercel deploy` resolves the target project from `.vercel/project.json`
in the current working directory. A worktree is a fresh git checkout
with no Vercel link copied over. The CLI's "first deploy?" interactive
prompt becomes non-interactive in scripted contexts → defaults to "create
new project" with a hash-derived name.

### Prevention (sub-agent prompts)

When briefing a sub-agent that will run `vercel deploy` from a worktree,
include this step in the deploy section:

```bash
# Copy parent's Vercel linkage so deploy targets the real project.
ls -la .vercel/ 2>/dev/null || cp -r "/Users/gob/projects/mooniex-webapp (Desk-X)/.vercel" .vercel
vercel deploy
```

Substitute `Desk-X` with the parent session's desk. Verify after deploy:

```bash
vercel ls mooniex-webapp | head -3
# the deployment URL should appear under mooniex-webapp, NOT under agent-*
```

### Cleanup (after the fact)

```bash
# Find orphan
vercel project ls | grep -E "agent-[a-f0-9]+"
# Delete (replace ORPHAN-NAME)
vercel project rm ORPHAN-NAME
```

Vercel auto-cleans projects with zero deploys after long inactivity, so
the orphan from 2026-04-28 (`agent-a235d108b61f5cdf6`) was already gone
by 2026-04-29 verification — but don't rely on that.

---

## Failure mode 2 — Hook deadlock from two same-hash sessions

### Symptom

Two Claude sessions run on the same Mac, both with cwd in
`mooniex-webapp (Desk-D)/`. The PreToolUse hook
(`~/.claude/hooks/office-pretool.mjs`) generates `agent_id` from
`sha256(${hostname}|${cwd}|${day}).slice(0,12)` — both sessions hash
to the same id. First session's `/api/office/claim` succeeds. Second
session's `/api/office/claim` returns 409, and `tryAdoptOrphan()`
either fails (heartbeat 404 if claim was already released by sweep)
or succeeds for the wrong cache file. The losing session sees:

```
Desk Desk-X is occupied by Claude · <hostname> · YYYY-MM-DD
since <timestamp>. Switch to a free desk before editing.
```

…blocking ALL `Bash`/`Edit`/`Write` until resolved. `cd` to a free
desk doesn't help because `cd` IS Bash — blocked at the first call.
Catch-22.

### Why

`tryAdoptOrphan()` heartbeats the existing claim_id, but `claim_id`
on the API side belongs to the OTHER session's cache. Heartbeat
succeeds (it's just an UPDATE on `last_heartbeat_at`), the losing
session writes its own cache pointing at someone else's `claim_id`,
and any subsequent action that depends on its own identity (MCP
`send_message` sender resolution, pre-existing claim ownership) is
inconsistent.

The `mcp__mooniex-coord__send_message` resolver looks up the cache
file at `~/.claude/office-state-<myCwdHash>.json` and uses its
`agent_id` as the sender. When the cache was written by adopt logic
pointing at a different session's claim, the MCP resolver fails to
match and returns `cannot resolve sender agent_id`.

### Prevention

1. **Do NOT run two Claude sessions on the same desk folder.** It's
   a hard collision. If two desks are needed, open the second one in
   a different `mooniex-webapp (Desk-X)` folder.
2. **Sub-agents via Agent tool — use `isolation: "worktree"`** so
   sub-agent's cwd is a temp path that doesn't match the desk regex
   `^mooniex-webapp \(([^)]+)\)$`. Hook no-ops; no claim attempted;
   no collision possible.

### Recovery (when the deadlock has already happened)

| Option | Steps | Time |
|--------|-------|------|
| **A. CEO-side release** | `/admin/office-simulator` → Desk-X → "Release". Losing session's next Bash call re-claims fresh. | 10 sec |
| **B. Wait for `office-sweep`** | Hourly cron (`0 * * * *`) releases claims with stale heartbeat (> 5 min idle). | Up to 1h |
| **C. API release** (CEO authorized) | One-shot script reads `claim_id` from `/api/office/state`, posts `/api/office/release`. Bearer auth via `OFFICE_TOKEN`. | 30 sec |
| **D. Switch desks** | Spawn `Agent` with `isolation: "worktree"` → sub-agent works in temp path → no claim conflict. | 1 min |

Option D was used 2026-04-28 to ship the cron dashboard hardening
while Desk-D was deadlocked.

### Long-term fix — shipped 2026-05-17

Hook script (`~/.claude/hooks/office-pretool.mjs`) `tryAdoptOrphan()`
now refuses adoption when the rival claim is fresh AND no local cache
exists. Backup at `~/.claude/hooks/office-pretool.mjs.bak.20260517`
(restore via `cp .bak.20260517 office-pretool.mjs` if regression).

Guard logic:
```js
const heartbeatFreshMs = 60_000;
const hbStr = cur.last_heartbeat_at;
const hbMs = hbStr ? Date.parse(hbStr) : null;
const isFresh = hbMs && now - hbMs < heartbeatFreshMs;
const cacheExists = (await readJson(CACHE_PATH)) != null;
if (isFresh && !cacheExists) {
  return false;  // let main() emit clean deny
}
```

Sessions with a local cache (i.e. the legitimate holder) skip the
guard. Brand-new second session arriving sees the existing deny path
("Desk-X is occupied by Claude · ... since YYYY-MM-DD HH:MM:SS") and
must switch desks or wait for the rival to release.

Still open (proposed, not shipped):
- Surface rival's hostname in the deny so the operator can identify
  which window to close. Requires resolving `cur.agent_label` server-side
  to extract the hostname segment.
- CI lint to detect when the hook script itself drifts from this
  playbook (e.g. someone removes the guard).

---

## See also

- [office-coordination.md](office-coordination.md) — desk model,
  `OFFICE_TOKEN`, claim/heartbeat lifecycle.
- [agent-messaging.md](agent-messaging.md) — `mcp__mooniex-coord`
  failure recovery (covers the "cannot resolve sender" subtype).
- [IRON-RULES §1](https://github.com/PASAKON/Agents-Wikis/blob/main/IRON-RULES.md#section-1--desk-model-one-agent-per-folder) (`org:IRON-RULES.md`) — desk model rule.
