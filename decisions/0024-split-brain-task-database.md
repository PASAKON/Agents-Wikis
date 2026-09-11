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

**Recommendation: option 2** — and the phone-control objection turns out to be
void, which was not known when this ADR was first written.

### Correction, same day: the relay already exists and already runs

The CEO asked whether the single database should live on Contabo instead, since
that box is always online. Checked, and the org already solved this a month ago
in the opposite direction (task-776fbf7e, task-18241f1d):

- `runners/relay_mcp_server.py` runs on Contabo and owns a `relay_queue` table.
- `runners/mac_agent.py` runs on the Mac as launchd job `com.mooniex.mac-agent`
  (**pid 659, live**) and drains that queue.

Its own module docstring states the constraint that settles the question:

> "Direction matters: **the Mac dials out**. Contabo cannot reach it — SSH is
> closed there, verified `Connection refused` — and that asymmetry is the
> security design, not a gap to fix."

So moving the database to Contabo would mean either opening the Mac to inbound
connections (giving up that security property) or building a network service in
front of SQLite, which cannot be shared over a network share safely. On top of
that, most repos simply are not on Contabo — `mooniex-webapp`, `cookierun-bot`,
`mooniex-remotion` and others live only on the Mac, so a worker there cannot edit
them at all.

**The right shape is the one already built:** Contabo is the always-on front door
that receives and queues; the Mac is the single hub that owns `tasks.db` and
executes. Option 2 therefore costs the CEO nothing — Contabo keeps the ability to
drive winbox, but through the relay instead of by ssh-ing there itself and
recording the task in a database nobody else can see.

## What must not be concluded from this

That the VPS database is junk to be deleted. It holds the only record of work that
really ran, and one of its rows describes a process still alive on winbox. Nothing
here is cleaned up without reading it first.
