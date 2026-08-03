# Wiki Maintenance — Ownership Rules

> **Style + structure** of wiki entries lives in [CONTRIBUTING.md](CONTRIBUTING.md). This file is about **who** owns what — see CONTRIBUTING for **how** to write.

## Who updates what

**Rule**: the agent (DEV) who edits a project's source rule is responsible for updating that project's wiki entry **in the same commit / push cycle**.

| Wiki file | Owned by |
|-----------|----------|
| `projects/mooniex-webapp.md` | DEV editing `mooniex-webapp/AGENTS.md` or `CLAUDE.md` (any desk) |
| `projects/mooniex-webapp-desk-{1,b,c,…}.md` | DEV who first sits at a renamed/new desk; thin per-desk file |
| `projects/mooniex-claudeflow.md` | DEV editing `mooniex-claudeflow/CLAUDE.md` or `docs/ops/AI_OPERATING_POLICY_V2_TH.md` |
| `projects/mooniex-option.md` | DEV editing `mooniex-option/setup-windows.md` or `CHANNEL_PLAN.md` |
| `projects/<other>.md` | DEV editing that folder's `CLAUDE.md` / `README.md` / setup docs |
| `IRON-RULES.md` | DEV who introduced the rule (must update its quoted section) |
| `CROSS-FOLDER-MAP.md` | DEV who added/changed/removed a cross-folder edge |
| `INDEX.md` | DEV adding/removing a project folder |
| `playbooks/*.md` | DEV who changes the underlying procedure (rare — coordinate with user first) |

## When to update

**Update the wiki BEFORE `git push`**, not after. The wiki ships with the rule it documents.

| Trigger | Action |
|---------|--------|
| Edit `CLAUDE.md` / `AGENTS.md` / `setup-windows.md` in any project | Patch matching `LLMs/projects/<that>.md` + relevant verbatim quote in `IRON-RULES.md` |
| Add new cross-folder dependency (webhook, env share, table FK) | Add edge to `CROSS-FOLDER-MAP.md` mermaid + per-edge detail section |
| Add new project folder under `/Users/gob/projects/` | Create `projects/<new>.md`, add row to `INDEX.md` |
| Delete project folder | Delete `projects/<old>.md`, remove row from `INDEX.md`, remove edges from `CROSS-FOLDER-MAP.md` |
| Postmortem of incident → new rule | Add to `IRON-RULES.md` + project file "Recent incidents" section |
| Quarterly (every 3 months) | User runs `/wiki-sync --full` to catch drift |

## What NOT to update

- Don't touch wiki for transient state (in-progress task, today's TODO)
- Don't paraphrase verbatim quotes — re-pull from source
- Don't update other projects' wiki entries unless you also edited their source rule

## Pre-push checklist (mandatory before every wiki push)

Run through this before `git push` from `/Users/gob/projects/LLMs/`:

```bash
cd /Users/gob/projects/LLMs

# 1. List staged + modified files
git status --short

# 2. For every projects/*.md edited, grep IRON-RULES.md for new rule keywords
for f in $(git diff --name-only HEAD | grep '^projects/'); do
  echo "=== $f ==="
  # extract bold "Rule:" / new headings / env vars added in this diff
  git diff HEAD -- "$f" | grep -E '^\+' | grep -iE '(rule|env|api_key|secret|prefix|require|must|never)' | head -5
done

# 3. If output above shows new rules → add them to IRON-RULES.md too in this same commit
# 4. New cross-folder edge mentioned? → CROSS-FOLDER-MAP.md
# 5. New / removed folder? → INDEX.md
```

**Why**: incident 2026-04-26 — DEV pushed `projects/mooniex-webapp.md` with 5 new rules (Skeleton loader, Sentry `event:write`, Fal admin-scope key, Cron header gate, new env vars) but did NOT update `IRON-RULES.md`. Other DEVs reading IRON-RULES missed them. Sync done in follow-up commit.

**Rule**: when a `projects/*.md` diff introduces text that looks like a rule (new env var, new HTTP header, new API requirement, new pattern, "must"/"never"/"required" wording), the same commit MUST patch `IRON-RULES.md`. Project-only facts (admin pages list, member surfaces, table list) stay in `projects/*.md` only.

## Diff workflow (incremental update)

```bash
# 1. After editing project's CLAUDE.md, diff against wiki
diff <(grep -A 30 "Hard rules" /Users/gob/projects/mooniex-webapp/AGENTS.md) \
     <(grep -A 30 "Hard rules" /Users/gob/projects/LLMs/IRON-RULES.md)

# 2. If diff > 0, patch wiki to match source
# 3. Commit both source + wiki together:
git add AGENTS.md
cd /Users/gob/projects/LLMs && git add projects/mooniex-webapp.md IRON-RULES.md
git commit -m "feat(rules): add X rule + sync wiki"
```

## Wiki repo (live)

Wiki is a git repo:
- **Local**: `/Users/gob/projects/LLMs/`
- **Remote**: `https://github.com/PASAKON/mooniex-llms-wiki`
- **Branch**: `main` (no other branches — too small to justify branching)
- **Auto-pull**: SessionStart hook in `~/.claude/settings.json` runs `git -C /Users/gob/projects/LLMs pull --ff-only --quiet 2>/dev/null || true` on every Claude Code session start

Concurrency rules for the wiki itself (Hot/Cold zones, conflict resolution): see `LLMs/CLAUDE.md`.

## When DEV doesn't have edit rights to wiki folder

Wiki = `/Users/gob/projects/LLMs/` — full edit by any Claude (no restrictions). All DEVs can patch directly. Read-only rules apply only to project folders.
