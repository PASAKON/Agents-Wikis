# Playbook: Org Operating Model (Agents/)

**Source of truth for how the agent organization at `/Users/gob/projects/Agents/` works.**

Created: 2026-05-18 (CEO directive)

---

## Roles

| Role         | Level  | Model            | Wiki write | Merge | Push |
|--------------|--------|------------------|:----------:|:-----:|:----:|
| CEO          | C      | (human)          | yes        | yes   | yes  |
| CTO          | C      | claude-opus-4-7  | yes        | yes   | yes  |
| frontend_dev | worker | claude-sonnet-4-6| no         | no    | no   |
| backend_dev  | worker | claude-sonnet-4-6| no         | no    | no   |
| devops       | worker | claude-sonnet-4-6| no         | no    | no   |
| qa           | worker | claude-haiku-4-5 | no         | no    | no   |

Permissions are enforced by tool gates in `tools/`. DEVs that try to push are
blocked at the tool layer, not by convention.

## Flow

```
CEO request
   │
   ▼
[CTO]  reads wiki, plans, creates tasks in SQLite queue
   │
   ▼   for each task:
[CTO]  create_worktree → branch agent/<role>-<task_id>
   │
   ▼
[DEV]  spawned subprocess, cwd = own worktree only
   │   reads wiki (read-only)
   │   reads codebase
   │   writes code + commits inside worktree
   │   produces structured report
   │
   ▼
[CTO]  reviews report + diff
   │   pass → merge to default branch → auto-push (per project config)
   │   fail → reopen task with feedback (max 3 iterations)
   │
   ▼
[CTO]  updates wiki on significant decisions
[CTO]  reports back to CEO
```

## Project Registry

Live at `config/projects.yaml`. Currently:

- `mooniex-claudeflow` — github.com/PASAKON/mooniex-claudeflow
- `mooniex-webapp` (Desk-A) — github.com/golfmaichai1/mooniex-webapp
- `LLMs` (the wiki itself, C-level write only) — github.com/PASAKON/mooniex-llms-wiki

Adding a new project = append to `projects.yaml`, no code change.

## Conflict Prevention

- **One task → one branch → one worktree → one DEV.** Never share.
- **DEVs never touch default branch.** `cwd` is locked to worktree.
- **DEVs never push, never delete branches.** Tool gates reject.
- **CTO merges serially per project** to avoid main-branch races.
- **Wiki writes** are also single-writer per session (CTO only).

## Observability

Three layers, terminal-first:

1. **TUI dashboard** — `python dashboard.py` (Textual, refresh 2s)
2. **tmux multi-pane** — `bash scripts/watch.sh` (4-pane CTO + queue + 2 DEV slots)
3. **Log files** — `state/logs/<role>_<task_id>.log` + `state/logs/<role>_latest.log` symlink

## Wiki Discipline (CTO writes)

- **Only when meaningful** — not chatter.
- **Date-stamp + attribute** every page edit.
- **Keep pages short and indexed.** Use `INDEX.md`.
- **ADRs** in `decisions/ADR-<NNN>-<slug>.md`.
- **Postmortems** in `postmortems/<date>-<slug>.md`.

## Quota Strategy (Max $200 subscription)

- CTO = Opus (smart orchestrator, fewer calls)
- Senior DEVs = Sonnet (most work)
- QA = Haiku (cheap)
- Cache wiki content (prompt caching).
- Cap parallel DEVs at 3 to avoid rate limits.

## Failure Modes + Recovery

| Failure                   | Detection             | Recovery                                       |
|---------------------------|-----------------------|------------------------------------------------|
| DEV subprocess crashes    | `delegate_task` rc!=0 | Task → `failed`, CTO investigates              |
| Merge conflict on main    | git merge fails       | CTO triggers rebase task or aborts            |
| Wiki write race           | Tool-level lock       | Single CTO at a time per session              |
| Worktree stale            | `git worktree prune`  | `scripts/cleanup.sh` (TBD)                    |
| Rate limit (Max sub)      | SDK error             | Backoff + queue                               |

## Not Yet Implemented (Phase 2+)

- Multi-CTO coordination across sessions
- Web dashboard
- Cost tracking per task
- Notification routing (Slack/Line)
- Cross-project dependency tasks
