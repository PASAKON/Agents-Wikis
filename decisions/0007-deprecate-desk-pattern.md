# ADR 0007 — Deprecate the multi-desk folder pattern

- **Status**: Accepted
- **Date**: 2026-05-28
- **Decider**: CEO (`pass.gob1@gmail.com`)
- **Implemented by**: CTO session `11d195af`

## Context

`mooniex-webapp` had been cloned into seven sibling folders under
`/Users/gob/Projects/` — `mooniex-webapp (Desk-A)` through `(Desk-F)` plus
`(Desk-CTO)`. Each "desk" was nominally owned by one agent role (Senior
Dev × 3, ML, DevOps, QA, CTO) so two agents could edit different files on
their own working copy and let GitHub be the merge engine.

The model worked when parallel agents lacked a stronger coordination layer.
It accumulated cost over time:

- **~5 GB disk** burned on duplicate `node_modules` / `.next` trees, none
  of which agreed on lockfile state for long.
- **Stale remote drift** — only `Desk-CTO` had been updated to the new
  GitHub user `PASAKON`; the other six still pointed at the renamed-away
  `golfmaichai1` (GitHub HTTP-redirects so the URLs still worked, but
  `git remote -v` lied about ownership).
- **Branch divergence** between desks: `Desk-A` was on `main`, `Desk-C`
  was on a feature branch, the rest were on `bootstrap/landing-mvp` with
  three different SHAs. None were ahead of origin, but the pile made it
  hard to answer "what is the current canonical state?"
- **Concurrency hook collisions** — the `PreToolUse` hook hashed
  `(hostname, cwd, day)` to derive `agent_id`. Two Claude sessions on the
  same desk deadlocked; running anywhere else (e.g. `/Users/gob/Projects/`
  root) silently no-op'd the hook.
- **Wiki overhead** — separate `projects/mooniex-webapp-desk-{a..f,cto}.md`
  files duplicated 80% of `projects/mooniex-webapp.md` and went stale.

Meanwhile parallelism is now achieved without separate working copies:

- The Agents virtual org (`/Users/gob/Projects/Agents/`) creates a fresh
  `git worktree` per DEV task under `worktrees/<project>__<role>__<task-id>/`,
  scoped per CTO + locked via the `tasks.touches` collision table.
- Branches + PRs handle the merge negotiation that desks used to handle
  by being on different folders.
- GitHub Issues now serialise hot-file edits (`src/middleware.ts`,
  `src/app/layout.tsx`, `package.json`, etc.) by single-assignee.

## Decision

**Deprecate the multi-desk folder pattern. One repo = one canonical folder.**

- `mooniex-webapp` lives at `/Users/gob/Projects/mooniex-webapp/` only.
- Parallel work uses branches and (when truly needed on one machine)
  `git worktree`, not duplicate clones.
- Conflict prevention moves to the GitHub layer:
  - Hot files require an open GH issue claim before edit (one assignee).
  - PRs review what the desks reviewed implicitly.
  - The Agents virtual org continues to lock at the `tasks.touches` level
    for in-flight DEV tasks.
- Roles are assigned per task, not per folder. The Agents
  `config/projects.yaml` `agents_allowed` list is authoritative.

## Consequences

### Positive

- ~5 GB disk reclaimed in the 2026-05-28 cleanup.
- One remote URL across the team; no "which clone has the right user?"
- One npm/pnpm install across the working copy. Lockfile drift gone.
- Wiki: one `projects/mooniex-webapp.md` instead of eight near-duplicates.
- Removes the deadlock failure mode where two Claude sessions land on the
  same desk.

### Negative / trade-offs

- Agents who genuinely need two checkouts (rare — usually a long-running
  build on one branch while editing another) must learn `git worktree add`.
  Documented in `IRON-RULES.md` §1.5 + `playbooks/worktree-and-concurrent-sessions.md`.
- Existing tooling that hard-codes `Desk-X` (e.g. the `webapp_office_agents`
  table, `/api/office/claim` endpoint, the office Simulator at
  `/admin/office-simulator`, the `office-pretool.mjs` hook) is now legacy.
  Retire on the next pass — see `MIGRATIONS.md` §desk-pattern.
- The `IRON-RULES.md` body still references `Desk-X` letters in §16/§17/
  examples; those passages are now legacy text. The §0 banner at the top
  of `IRON-RULES.md` flags this until a full sweep lands.

## Migration steps (executed 2026-05-28)

1. Survey: 7 desk folders, none ahead of origin; dirty content all tooling cache.
2. Rename `mooniex-webapp (Desk-CTO)/` → `mooniex-webapp/` (the only desk
   already on the new PASAKON remote).
3. Delete the other 6 desk folders.
4. Update `Agents/config/projects.yaml` path + remote.
5. Update `Projects/CLAUDE.md` + `~/.claude/CLAUDE.md` path references.
6. Rewrite `IRON-RULES.md` Section 1.
7. This ADR.
8. Mark stale wiki: delete `projects/mooniex-webapp-desk-{a..f}.md`, add
   `MIGRATIONS.md` entry.

## Open follow-ups (deferred)

- Retire `webapp_office_agents` table + `/api/office/claim` route in `mooniex-webapp`.
- Retire `office-pretool.mjs` hook OR refactor to use branch-based agent id.
- Strip legacy `Desk-X` mentions from `IRON-RULES.md` §16+§17 bodies once
  the office infrastructure is removed (this is a wiki-only edit, not
  blocking).
- Update `roles/` markdown in `Agents/` to drop desk references.
