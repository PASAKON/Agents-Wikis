# LLMs Wiki — concurrency rules

This folder is the **shared knowledge base** for every Mooniex DEV (Claude / Codex agent). It lives on every machine via `git pull`, and is read by every project's `CLAUDE.md` via `@import`.

## Read

Before any work in any `mooniex-*` folder:
- `INDEX.md` — project matrix
- `projects/<that-folder>.md` — your project's facts
- `IRON-RULES.md` — canonical rules (verbatim)
- `playbooks/before-cross-folder-edit.md` — if editing another folder

## Hard rules (verbatim from `mooniex-webapp/AGENTS.md`, applied to wiki)

1. **Pull before edit, every time.** `git -C /Users/gob/projects/LLMs pull --ff-only` before any change. The SessionStart hook does this automatically — but verify with `git status` if you've been idle.
2. **Push immediately after commit.** Never leave wiki commits sitting locally — other DEVs won't see your update until you push.
3. **Append > rewrite in shared files.** `INDEX.md`, `IRON-RULES.md`, `CROSS-FOLDER-MAP.md` are written by every DEV. Add new sections / rows / edges instead of rewriting existing ones, so merges stay clean.
4. **Conflict resolution**:
   - `projects/<your>.md` collision: rare (only one DEV owns each) — hand merge
   - `INDEX.md` / `IRON-RULES.md`: take both rows / sections, dedupe later
   - Mermaid edges in `CROSS-FOLDER-MAP.md`: keep both, hand merge

## Hot Zones (serialize — one DEV at a time)

| File | Why |
|------|-----|
| `IRON-RULES.md` | Canonical rules — every DEV reads it; rewrite breaks others' mental model |
| `INDEX.md` | Project matrix — narrow file, easy to collide |
| `CROSS-FOLDER-MAP.md` | Mermaid diagram — text-merge unfriendly |
| `MAINTAINERS.md`, `README.md` | Meta — change requires user approval |

**Rule**: if 2 DEVs need to edit a Hot Zone file at the same time → second DEV waits.

## Cold Zones (parallel OK)

| Files | Why |
|-------|-----|
| `projects/<name>.md` | Each owned by the DEV editing that project's source rule — no collision |
| `playbooks/*.md` | Rarely edited; coordinate via user if needed |

## Update flow (when you change a project's source rule)

```bash
# 1. Pull wiki (covered by SessionStart hook, but explicit is fine)
git -C /Users/gob/projects/LLMs pull --ff-only

# 2. Edit your project + matching wiki entry in same change set
cd /Users/gob/projects/<your-project>
$EDITOR CLAUDE.md  # or AGENTS.md / setup-windows.md

cd /Users/gob/projects/LLMs
$EDITOR projects/<your-project>.md
$EDITOR IRON-RULES.md  # if rule is in there

# 3. Commit + push wiki FIRST
git add . && git commit -m "sync: <project> rule X"
git push origin main

# 4. Then commit + push project
cd /Users/gob/projects/<your-project>
git add CLAUDE.md
git commit -m "feat(rules): X + sync wiki"
git push origin <branch>
```

Wiki ships **before** the project change — so other DEVs reading wiki on next session-start see the rule before they encounter the code.

## Skip wiki update for

- In-progress task state, today's TODOs
- Pure code changes that don't change a documented rule
- Bug fixes that don't change the rule itself

## Auto-pull hook

`~/.claude/settings.json` SessionStart hook runs:
```bash
git -C /Users/gob/projects/LLMs pull --ff-only --quiet 2>/dev/null || true
```

Silent fail if offline or no remote — agent falls back to cached wiki.

## Repo

- **Remote**: `https://github.com/PASAKON/mooniex-llms-wiki`
- **Branch**: `main` (no other branches — too small a repo to justify branching)
- **Push policy**: any DEV can push directly to `main`. Pull-before-edit + push-immediately = safe enough at this scale.
