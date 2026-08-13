# ADR 0021 — One test convention, real isolation, and five critical paths

- **Date:** 2026-08-13
- **Status:** Accepted
- **Decider:** CEO
- **Source:** Hermes Agent survey — `AGENTS.md` §Testing; measured state of our own suite

## Context

The org runtime has **43 files named `test_*.py`** (39 under `scripts/`, 4
under `lib/`). It also has, measured 2026-08-13:

| Check | Result |
|---|---|
| `pytest` installed in `.venv` | **No** — `No module named pytest` |
| `pytest.ini` / `pyproject.toml` / `tox.ini` / `Makefile` | **None** |
| `.github/workflows/` | **Does not exist** — no CI |
| Convention | **Three at once**: 9 `unittest`-based, ~33 with bare `def test_`, several documented as `python scripts/test_x.py` standalone |

So: 43 test files that nobody can run with one command, that no automation ever
runs, written three different ways. That is worse than having no tests. Untested
code is a known risk; unrunnable tests are a *believed* safety net that is not
there.

### The isolation failure, found by trying

Running the obvious discovery command:

```
python -m unittest discover -s scripts -p 'test_*.py'
```

did not report test results. It **executed a real evaluation pipeline** — a
caption A/B eval printing `win=2 lose=0 tie=16`, ending with:

```
[proof] wrote /Users/gob/Projects/Agents/output/trader-mindset/_test/sample_composite.png
```

Test discovery wrote a PNG into the CEO's real output tree.

The cause is that `scripts/test_*.py` contains two different kinds of file
under one naming convention: genuine unit tests, and eval/verification harnesses
that do real work with real side effects. Discovery cannot tell them apart,
because nothing distinguishes them.

This is precisely the failure Hermes wrote a rule against: *"tests must not
write to `~/.hermes/`."*

## Decision

### 1. Adopt Hermes' three anti-patterns verbatim

**Do not write change-detector tests.** A test that asserts a count, a catalog
snapshot, or a version literal goes red on every unrelated addition and teaches
everyone to ignore red. Ban list, with our equivalents:

```python
# BAD — breaks whenever a skill is added
assert len(list_skills()) == 19
# BAD — breaks on every model release
assert DEFAULT_MODEL == "claude-sonnet-5"
# BAD — breaks on every schema bump
assert SCHEMA_VERSION == 7

# GOOD — behaviour: does the loader work at all?
assert load_skills(), "skill loader returned nothing"
# GOOD — invariant: every configured role resolves to a known model
for role in ROLES: assert resolve_model(role) in KNOWN_MODELS
```

**Tests must not write to real state.** No test may touch `state/tasks.db`,
`state/locks/`, `output/`, `~/.claude/`, or any wiki root. Fixtures use
`tmp_path`. Anything needing a DB builds a throwaway one.

**Do not fake the host OS.** If it only passes on a simulated platform it does
not tell us the Mac works.

### 2. One convention, one runner

- **pytest.** Standardise on it; `pytest` is added to `requirements.txt`.
- **`pytest.ini`** at the repo root, `testpaths = scripts lib`, and a marker
  registry.
- Existing `unittest`-based files keep working (pytest runs them); no rewrite
  campaign, convert on touch.

### 3. Evals move out of the test namespace

Files that perform real work with real side effects are not tests. They move to
`scripts/evals/` and lose the `test_` prefix. If one must stay in place, it is
marked `@pytest.mark.eval` and excluded from the default run.

This is the fix for the PNG incident, and it is a precondition for anything
else being safe to run.

### 4. Five critical paths get real coverage

Chosen because each has already broken in production and each is load-bearing:

| # | Path | Why |
|---|---|---|
| 1 | `create_task → delegate_task → submit_report → merge_task` | The org's main loop. Nothing covers it end to end. |
| 2 | `touches` collision + `locks` contention | Two DEVs on overlapping paths is the failure `check_collisions` exists to prevent. |
| 3 | `merge_task` must verify `main` actually advanced | Known bug: merge can no-op, delete the branch, and report `merged=true`. `test_merge_noop_guard.py` covers the git primitive; the tool-level path is uncovered. |
| 4 | Worktree lifecycle — create, reap, cleanup at `merged` | Audit W6 (2026-08-07) found worktrees stuck at `merged` falling between both cleanup mechanisms. |
| 5 | DEV resume / re-delegate does not silently die | Repeated incident: re-delegate skips `set-pending`, claim fails, DEV dies quietly. |

### 5. CI

A single GitHub Actions workflow on `mooniex-agents`: run pytest, run
`osv-scanner` ([ADR 0020](0020-runtime-self-protection-guards.md)), run
gitleaks. Green required before `merge_task` on this repo.

## Consequences

- The runtime becomes changeable without fear, which is the precondition for
  [ADR 0018](0018-skill-lifecycle-and-curator.md),
  [ADR 0019](0019-memory-self-nudge-and-session-search.md), and
  [ADR 0020](0020-runtime-self-protection-guards.md) — all three modify it.
- Some currently-passing files will fail once they can actually run. That is
  information we do not have today and want.
- `scripts/` gets smaller and more honest as evals move out.
- CI is the one piece of automation this work adds, and it is deterministic —
  fully consistent with [ADR 0017](0017-no-autonomous-task-dispatch.md), which
  bans autonomous *agents*, not scripts.
