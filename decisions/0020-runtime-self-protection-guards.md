# ADR 0020 — Code-enforced guards: no agent edits the runtime it is running on

- **Date:** 2026-08-13
- **Status:** Accepted
- **Decider:** CEO
- **Source:** Hermes Agent survey — `tools/self_repo_guard.py`, `tools/osv_check.py`

## Context

Our security model and Hermes' are on **different axes**, which is why they
merge cleanly rather than compete.

| | What it controls | Examples |
|---|---|---|
| **Ours** | *Who is allowed* | GateGuard fact-forcing, role flags (`can_merge` / `can_push` / `can_write_wiki`), `policies/permissions.md`, CEO approval before paid APIs, API key registry (IRON §34), gitleaks pre-commit + CI |
| **Hermes'** | *What is dangerous* | `path_security.py`, `url_safety.py`, `threat_patterns.py`, `write_approval.py`, `osv_check.py`, `self_repo_guard.py`, DM pairing, container isolation |

Authority versus danger. Neither substitutes for the other. Ours is
process-enforced and depends on a human being in the loop reading a prompt;
theirs is code-enforced and holds when nobody is watching.

Two of their guards cover holes we have actually fallen into.

### `self_repo_guard`

Hermes documents the failure plainly in its own contributor guide: a venv
inside the directory the agent operates from "can be wiped by a relative-path
command the agent runs against its own checkout, destroying the running runtime
mid-session."

We have the sharper version of this exposure. DEVs run in
`worktrees/mooniex-agents__<role>__task-XXXX/` — **git worktrees of the org
runtime itself**. A DEV given a task in `mooniex-agents` is editing the code
that spawned it, the code that will review it, and the code that will merge it.
Existing memory records related incidents: `git add -A` smuggling destructive
diffs into a merge (2026-07-18), and `merge_task` deleting a branch while
merging nothing yet reporting success.

Nothing in the runtime currently refuses a write to a load-bearing path. The
only thing standing between a confused DEV and a broken org is that the DEV
usually behaves.

### `osv_check`

`requirements.txt` and the various `package.json` files are never scanned for
known CVEs. gitleaks covers *secrets committed by us*; it says nothing about
*vulnerabilities shipped to us*. The security audit of 2026-07-23 found no key
leak and installed gitleaks 8.30 across three repos, but dependency CVEs were
out of its scope.

## Decision

### 1. `self_repo_guard` — a `PreToolUse` hook on Edit / Write / Bash

Refuses, from any DEV worktree, mutation of the load-bearing runtime:

- `lib/`, `runners/`, `tools/`, `policies/`, `config/`
- `.claude/settings.json`, `scripts/hook-*.py`
- `state/tasks.db` and anything under `state/locks/`
- any `.venv/` or `venv/` directory
- the `.git` directory of a parent checkout

**Escape hatch, deliberately narrow:** a task whose `touches` explicitly names
the path is allowed. This is the whole point of `touches` — the C-level
declared the intent up front, at `create_task` time, before any agent was
running. An undeclared write to a load-bearing path is by definition
unplanned, and unplanned writes to the runtime are the failure mode.

The guard fails **closed**: if it cannot determine whether a path is protected,
it refuses.

### 2. `osv_check` — dependency CVE scan

`osv-scanner` over `requirements.txt` and every `package.json`, wired into the
same CI job as the test suite ([ADR 0021](0021-testing-doctrine-and-critical-paths.md)).
Reports; does not auto-upgrade. Upgrading a dependency is a decision, and
decisions belong to a human.

### 3. Everything we already have stays

GateGuard, role capability flags, gitleaks, the API key registry, and the
paid-API stop are unchanged. This ADR only adds; it removes nothing.

## Consequences

- A DEV can no longer silently break the runtime that is supervising it. The
  blast radius of a confused agent drops from "the org" to "its own worktree".
- Some legitimate work will be refused until its `touches` is declared
  correctly. That is the intended pressure: declare intent before acting. It
  will be mildly annoying and should stay that way.
- Fail-closed means a bug in the guard blocks work rather than allowing it.
  Accepted: a false block is visible and loud, a false allow is silent.
- CVE findings will surface work nobody scheduled. They land as reported
  findings for a human to triage, not as automatic upgrades.
