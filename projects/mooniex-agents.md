# mooniex-agents (orchestration meta-repo)

Repo: `/Users/gob/Projects/Agents` · GitHub `PASAKON/mooniex-agents` · default branch `main` (direct push allowed, CTO merges).

The virtual-org runtime: CEO → CTO → DEV orchestration, task queue (`state/tasks.db`), iTerm visible-chat plumbing, MCP servers, watchdogs.

## Changelog

### 2026-06-12 — DEV-report routing hardened (issue #15) — CTO 37ad539a
- **Problem:** tasks created by non-CTO CXO sessions (CFO etc.) had `owner_cto=NULL` → DEV reports broadcast into EVERY open CTO tab (`send_to_cto` legacy path). Wrong CTO nearly merged foreign work; stay-in-lane rule was the only guard.
- **Fix (merge `4beb8d9`, task-2094c430):**
  - `lib/db.py` stamps `owner_cto` from `CTO_SESSION_ID` → `CXO_SESSION_ID` fallback; new additive `owner_role` column; loud stderr warning on ownerless create.
  - `tools/send_to_cto.py`: ownerless messages now orphan to `state/orphan-dev-replies-unowned.log` — **no broadcast** unless `SEND_TO_CTO_BROADCAST=1`. Winid lookup is role-aware (`<role>-<id>.winid`), so CFO-spawned DEV reports route to the CFO tab.
  - `DEV_CTO_ROLE` threaded through `dev_init` → `dev_mcp_server` / `hook-log-dev-reply`.
  - `scripts/backfill_owner_cto.py` — manual stamping of in-flight ownerless rows (`--list`, `--task --owner [--role]`). No auto-attribution by design.
- **Operational notes:** running C-level sessions must restart to pick up the change; lane owners backfill their own in-flight tasks (e.g. CFO lane task-1112d9d7).

### 2026-06-12 — paste-stall fixed (merge `0c4188b`, task-0d5f21e2) — CTO 37ad539a
- **Problem:** multi-line messages typed into claude TUI tabs stuck in the composer unsubmitted — body + CR written back-to-back, CR swallowed by bracketed paste (CEO screenshot 2026-06-12, recurring).
- **Fix:** shared `lib/iterm_type.type_submit_fragment()` — body → `delay 0.4` → CR → `delay 0.3` → rescue CR (no-op if first submitted). All 9 typewriter sites routed through it: send_to_cto/send_to_dev/send_to_cxo/inject_prompt via helper, idle-ping-watcher.sh + cxo-claude.sh inlined. New suite `scripts/test_iterm_typewriter.py` (8 tests). 47/47 green.
- **Operational note:** running watchers/sessions use the old code until restarted.

### 2026-08-06 — Full architecture audit (CEO-requested) — CTO
- **Ask:** find weaknesses to fix, strengths to keep, excess to cut/merge (incl. debug/perf-check tooling), token-reduction levers, skill/rule additions.
- **Method:** 4 parallel read-only Explore agents, ground-truth verified (not memory-derived) — core engine, scripts/tools/roles, disk-bloat root-cause, wiki+skills+hooks token footprint.
- **Top findings:** 3 drifting orchestration-tool surfaces (CEO's own `agents-chat` REPL missing `revert_task_tool`); `depends_on` still unenforced at the lock layer (matches existing CTO workaround-by-hand); worktree GC never calls `remove_worktree` → ~5GB of 5.3GB `worktrees/` is stale; debug/perf layer itself is proportionate and mostly silent (not the bloat source); biggest token lever is ~150-200+ irrelevant `ecc:*` skill packs enumerated every session despite being CLAUDE.md-flagged as hard-skips (needs plugin-level fix, not doc-level).
- **Full findings + evidence:** `org:reference/2026-08-06-agents-system-audit.md`.

### 2026-08-06 — Audit remediation wave 1: worktree reap + critical orchestration-gap fixes — CTO
- **task-2d7bc187** (merge `cdeffb3`, commit `89d7529`): `tools/gc_stale_tasks.py` now reclaims worktrees on terminal status (done/cancelled/stalled/failed), wired into all 3 existing stale-task loops + a new `--reap`/`--go` CLI mode (dry-run by default) for the pre-existing backlog. Deleted confirmed-dead `runners/dev.py` and `lib/db.py`'s `is_session_busy`. **Validated live**: both this task's own worktree and task-b5db982c's were reclaimed automatically the moment their merges landed — 0 manual cleanup needed.
- **task-b5db982c** (merge `0d7f916`, commit `c063a54`): closed audit findings W1 (the `agents-chat` REPL was missing `check_collisions`/`recall`/`reflect`/`revert_task_tool` — now full 17/17 tool parity with `cto_mcp_server.py`) and W2 (cross-CTO ownership guard existed only in `cto_mcp_server.py`; extracted to shared `lib/task_ownership.py`, `cto_mcp_server.py` refactored to use it — pure extract, zero behavior change — and `cto.py`'s `t_delegate`/`t_merge`/`t_reopen` now gated too, closing the gap for `main.py`'s one-shot mode and the REPL). Reviewed by reading the actual diffs/files (not just trusting the DEV report), re-ran the full test suite through `.venv` post-merge as an independent check.
- **Side-quest found during merge**: `merge_task`'s dirty-base guard correctly refused both merges — `state/locks/maintab-daemon.pid(.spawnlock)`, `state/locks/*.uuid`, `state/watchdog.log`, `state/fomo-scanner.db(+wal/shm/journal)` were untracked-but-unignored, so the live maintab daemon continuously re-dirtied the tree. Fixed `.gitignore` (commit `c181983`) — merged cleanly with a pending CEO edit to the same file (same intent: also excluding fomo-scanner.db, different syntax; kept the union). CEO's other ~68 pre-existing uncommitted files (untouched, unrelated) were stashed and restored byte-for-byte around the merge window — verified via daemon-file live-timestamp check post-restore.
- **Also applied**: `~/.claude/settings.json` `skillOverrides` — 76 irrelevant `ecc:*` domain-pack skills (kotlin/cpp/rust/go/swift/flutter/csharp/fsharp/perl/laravel/django/quarkus/springboot/nestjs/angular/nuxt4/dotnet/scientific/healthcare/logistics/network/homelab/defi/evm/etc.) set to `"off"`, matching CLAUDE.md's existing "hard skip" list. Global (all sessions/roles), reversible, doesn't touch python-review/typescript-reviewer/postgres-patterns. Takes effect next session — not verifiable live in the session that wrote it. Also moved `.asset-cache/` (72MB, dead Video-Engine leftover), `testsprite/`, `demo-api/` to `~/.Trash` (reversible, not `rm -rf`).
- **Still open**: the full "one manifest, one tool-surface" structural consolidation (audit's bigger recommendation behind W1/W2) — deliberately deferred, separate follow-up.
