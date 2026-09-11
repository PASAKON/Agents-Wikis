# ADR 0024 — The org runs two task databases, and neither knows about the other

- **Date:** 2026-09-11
- **Status:** Accepted (detection now; the structural fix is a CEO decision)
- **Owner:** CTO
- **Supersedes / relates to:** ADR 0013 (wiki namespace split), `2026-07-12-org-runtime-to-contabo.md`, IRON §50

## Context

Two `claude.exe` worker processes were found alive on winbox whose task ids —
`task-e44097d8` (pid 16352) and `task-34f1af72` (pid 22072) — **did not exist in
`state/tasks.db` on the Mac.** Nothing in the org could explain them.

They are not corrupt rows and not a bug in the spawner. They live in a **second
database**:

```
/opt/mooniex-agents/state/tasks.db        (Contabo VPS) — 4 rows
  task-34f1af72  browser_operator  in_progress  winbox  pid=22072
  task-e44097d8  browser_operator  cancelled    winbox  pid=16352
  task-41fa90ff  browser_operator  cancelled    winbox  pid=2864
  task-43c3f075  browser_operator  failed       winbox  pid=26472
```

All four carry `owner_cto = e1e3d3ef`, `owner_role = cto` — a CTO session running
on the VPS, live at the time of writing.

### Why this happens

`lib/db.py` resolves the database from **its own file location**, with no
environment override anywhere in the module:

```python
ROOT = Path(__file__).resolve().parent.parent
DB_PATH = ROOT / "state" / "tasks.db"
```

`_connect()` then does `DB_PATH.parent.mkdir(parents=True, exist_ok=True)` before
connecting, so a checkout that has no database **silently creates an empty one**
rather than failing. `state/*.db` is gitignored, and
`scripts/contabo-org-runtime-deploy.sh` explicitly excludes `state/`, so the VPS
necessarily starts from an empty database and fills it independently.

On the Mac this is normally masked: `config/cto.mcp.json` and
`config/worker.mcp.json` hardcode `/Users/gob/Projects/Agents` as `cwd`, so even a
worker in a git worktree talks to the canonical database. **That protection is
deliberately removed on the VPS** — `scripts/cto-claude.sh` regenerates the MCP
config at launch with `--root "$ROOT"`, precisely because the committed config
bakes in the Mac path and would otherwise break on a box where
`ROOT=/opt/mooniex-agents`.

So the VPS CTO writing to the VPS database is the system working exactly as
built. The defect is that **both boxes then ssh into the same winbox**, and
neither can see the other's workers.

## Consequence

- A VPS-spawned winbox worker is invisible to the Mac: no row, no pid, nothing to
  reap. It runs until someone notices it by hand.
- Neither box polls its own remote rows either. `runners/branch_poller.py` exists
  and is correct but is **scheduled by nothing** — no launchd plist, no cron, no
  systemd unit, and no running process on either machine. The only org daemon is
  `com.mooniex.agents-watchdog` on the Mac.
- Every existing recovery tool is **row-driven**: `runners/watchdog.py`,
  `tools/worker_reap.py`, `tools/gc_stale_tasks.py` and
  `scripts/session_orphans.py` all start from a task row. None enumerates
  processes on a host and asks "does this task id exist in my database?"
- The two boxes can therefore exceed winbox's `max_workers` without either
  noticing, and can in principle spawn two workers onto the same repo paths with
  no collision check between them.

## Decision

**Now (this session):** detect, do not kill. The CEO's instruction on the two live
orphans was *"ปล่อยไว้ก่อน ทำระบบให้ตรวจจับได้"* — leave them running, make the system
able to see them.

A report lists any worker process on a remote host whose task id is absent from
the local database. The identity test already exists and is proven: the Windows
launcher discovers a worker's pid by matching the task id inside `claude.exe`'s
command line, so the same query answers "whose is this?" from either side.

**Deferred to the CEO — the structural fix.** Three options, in increasing cost:

1. **One database, reachable from both boxes.** The Mac's `state/tasks.db` becomes
   the single authority and the VPS reaches it over the network. Correct, and the
   largest change: SQLite over a network share is not acceptable, so this means a
   real service in front of the database.
2. **Partition the hardware instead.** A box belongs to exactly one hub. The VPS
   stops spawning onto winbox; winbox belongs to the Mac. Cheapest, immediately
   enforceable in `config/hosts.yaml`, and costs the VPS its ability to drive the
   Windows machine from mobile Console sessions.
3. **Federate.** Each hub keeps its own database and publishes its remote rows
   where the other can read them. Preserves both capabilities and adds a sync
   path that can itself go stale, which is the failure mode this ADR is about.

**Recommendation: option 2**, unless the CEO specifically wants to keep driving
winbox from a phone. It removes the class of bug rather than managing it, and it
is a config change rather than a new service.

## What must not be concluded from this

That the VPS database is junk to be deleted. It holds the only record of work that
really ran, and one of its rows describes a process still alive on winbox. Nothing
here is cleaned up without reading it first.
