# Contributing to the Mooniex LLMs Wiki

Style + structure rules for writing in `/Users/gob/projects/LLMs/`. Read before adding or editing files. Goal: every agent's contribution looks like one author wrote it.

Companion files:
- [MAINTAINERS.md](MAINTAINERS.md) — who owns what, when to update
- [MIGRATIONS.md](https://github.com/PASAKON/MoonieX-Wikis/blob/main/MIGRATIONS.md) (`mooniex:MIGRATIONS.md`) — log breaking changes here
- [IRON-RULES.md](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md) (`mooniex:IRON-RULES.md`) — the canonical rules themselves

## 1. Folder layout

```
LLMs/
├── README.md                  entry point — links to everything
├── INDEX.md                   one-line per project (matrix)
├── IRON-RULES.md              canonical rules, verbatim quotes, numbered sections (§1, §2, …)
├── CROSS-FOLDER-MAP.md        mermaid + per-edge contracts
├── MAINTAINERS.md             ownership matrix + pre-push checklist
├── MIGRATIONS.md              disruptive-change log (newest first)
├── CONTRIBUTING.md            (this file)
├── DEV-PROMPT.md              copy-paste block for project CLAUDE.md
├── DEV-PROMPT-INLINE.md       same, ready for `@import`
├── projects/<name>.md         one file per /Users/gob/projects/* folder
└── playbooks/<name>.md        step-by-step for cross-folder ops
```

### Where does new content go? — decision tree

| New content | Goes in |
|-------------|---------|
| New `/Users/gob/projects/<X>/` folder appears | `projects/<X>.md` (new) + row in `INDEX.md` |
| New rule that applies across projects | New numbered section in `IRON-RULES.md` (`§N`) |
| New rule that applies to ONE project only | "Iron rules" section inside that `projects/<X>.md` |
| New cross-folder dependency (webhook, env share, table FK) | edge in `CROSS-FOLDER-MAP.md` |
| New step-by-step procedure used multi-project | `playbooks/<name>.md` (new) |
| Disruptive rename / schema break / rule reversal | NEW entry at top of `MIGRATIONS.md` |
| Style / structure rule for the wiki itself | append to this file |

If you can't tell where it goes, default to a `projects/<X>.md` section — local first, promote to IRON-RULES later if the rule generalises.

## 2. File naming

- **Lowercase + kebab-case** for new files: `mooniex-webapp.md`, `office-coordination.md`
- **Top-level reference files** are UPPERCASE: `IRON-RULES.md`, `MAINTAINERS.md`, `INDEX.md`, `MIGRATIONS.md`, `CONTRIBUTING.md`, `README.md`, `CROSS-FOLDER-MAP.md`, `DEV-PROMPT*.md`. Don't add new uppercase files without proposing in MIGRATIONS.md first
- **Per-project files** mirror the folder name: `mooniex-<name>.md`. Sub-variants (brand, roles, etc.) get a hyphen: `mooniex-webapp-brand-truth.md`, `mooniex-webapp-roles.md`
- **Playbooks** describe the action: `before-cross-folder-edit.md`, `database-migrations.md`, `vercel-deploy-coordination.md` — verb or topic, not the actor

## 3. Headings

- **`#` H1**: one per file, matches the file's purpose. Project files → folder name. Playbooks → "Playbook — <topic>". IRON-RULES sections → `## Section N — <topic>` (no H1 below the file's H1).
- **`##` H2**: top-level sections inside the file
- **`###` H3**: subsections; avoid going deeper than H4
- Keep heading text short — 4–8 words. Long sentences belong in the body

## 4. Section template — `projects/<name>.md`

Required sections, in this order. Skip a section by leaving it as `None` — never delete the heading:

```markdown
# <project-folder-name>

## Purpose
1–2 sentences. What this folder does, who runs it, what URL/binary ships.

## Tech stack
Bulleted: framework + key deps + runtime + build tool.

## Git
- Remote
- Branch
- Push policy (with cross-link to IRON-RULES §2 if relevant)

## Deploy
Target + URL + procedure (link to playbook, don't duplicate).

## Edit rights
"Full edit" / "Read-only" / "Per-desk" + cross-link to IRON-RULES §1.

## Env / secrets
Required + optional vars. Where they live (Vercel, .env.local, .env.enc).

## Database / Supabase
Project ref + table prefix + migration pattern.

## Iron rules (cross-references, not duplicated)
Bulleted list of IRON-RULES §N that apply.
Project-specific rules go inline below; cross-cutting rules link out only.

## Cross-folder relationships
Bulleted edges with arrows ←/→/↔ + cross-link to CROSS-FOLDER-MAP.

## Critical files
Top 3–7 file paths with one-line each.

## Recent incidents
Dated bullets. Include mitigation.

## Last commit (snapshot)
`<hash> <subject>` — wiki snapshots drift; readers should `git log -1` to confirm.
```

## 5. Section template — `playbooks/<name>.md`

```markdown
# Playbook — <topic>

One-line summary of when to run this.

Full rules: link to relevant IRON-RULES §N.

## One-time setup
Numbered steps. Token provisioning, env vars, prerequisite migrations.

## Daily flow
Numbered steps the agent runs every time.

## Failure modes
| Symptom | Cause | Fix |
|---------|-------|-----|

## What NOT to do
Bulleted footguns.
```

## 6. Style rules (every file)

### Verbatim quotes

When quoting a project's `CLAUDE.md` / `AGENTS.md` / `setup-windows.md`, **quote verbatim**. Use `> ` blockquote. Don't paraphrase — if the source moved, the wiki is stale and readers need the literal text to diff.

### Links

- **Internal links** = relative paths from the file's location: `[IRON-RULES §1](../IRON-RULES.md#section-1-…)` from `projects/`, `[wiki entry](mooniex-webapp.md)` within `projects/`
- **External links** = full URL
- **Project paths** in prose = backticked code: `` `/Users/gob/Projects/mooniex-webapp/` `` (canonical case = capital `P`, no `(Desk-X)` suffix)

### Tables

- One row per concept. Don't pack 5 things in one cell — use a sub-bullet list under the table or split rows
- Use ` | ` cell separators with `--- | --- |` header rule
- Right-align numeric columns is not required — readability over precision

### Code blocks

- Always specify language: ` ```bash `, ` ```ts `, ` ```sql `, ` ```json `
- Bash commands: prefer copy-paste-able. Hardcode paths only when they're stable (e.g. `/Users/gob/projects/...`)
- TypeScript: match the codebase's actual patterns (`logError`, `safeRoute`, etc) — wiki samples will be searched + adapted

### Lists

- Numbered when order matters (steps in a playbook, hard rules)
- Bulleted when it's a set
- Indent sub-items with 2 spaces

### No emoji (§14)

Wiki = source files. Same rule as the codebase: no emoji in `src/` extends to `LLMs/`. Use lucide-react icon names in prose if you need to refer to UI iconography (`<Check>`, `<X>`, `<CircleAlert>`).

`scripts/lint-emoji.mjs` in the webapp repo doesn't scan the wiki, but the spirit holds.

### No bare `console.error` in code samples

Same rule as §13. Sample code in the wiki uses `logError()` / `logErrorClient()` / `safeRoute()`. Bare `console.error` in samples = developers will copy + ship.

### Caveman mode

Wiki is **not caveman**. Wiki is read by humans + by AI agents who need full context to act. Write articles + complete sentences in the wiki, even when the user asks you in caveman mode.

## 7. Linking — mandatory inbound edge (CEO directive 2026-04-28)

**Every new file MUST have at least one inbound markdown link from an existing wiki file, committed in the same change set.**

A file with zero inbound links is invisible in the Obsidian graph view, falls outside the §0 "read-the-wiki-first" path (agents who don't already know the filename can't find it), and effectively wastes the writing effort. This is a hard rule, not a suggestion.

### Where to add the inbound link

| New file | Add inbound link from (at minimum) |
|---|---|
| `projects/<new>.md` | new row in `INDEX.md` AND a `CROSS-FOLDER-MAP.md` edge if it interacts with another project |
| `playbooks/<topic>.md` | the most relevant `IRON-RULES.md` section (e.g. error → §13, env → §5, deploy → §3, messaging → §17) AND every `projects/<x>.md` that uses the procedure |
| New `IRON-RULES.md` section | the section number is implicit in the file; also add a one-line entry to `MIGRATIONS.md` when numbers shift |
| New section inside an existing file | no extra link required — the parent file's existing inbound edges cover it |

### Backlinks for sibling docs

If a new playbook depends on or extends another playbook (e.g. `news-automation` builds on `content-cms`), **link forward AND back** — both files mention each other. Otherwise the topic graph only flows one direction and readers miss the prerequisite.

### Use real markdown links — not code spans

```markdown
GOOD:  See [`playbooks/secrets-rotation.md`](playbooks/secrets-rotation.md) for cadence
BAD:   See `playbooks/secrets-rotation.md` for cadence
```

The first creates a graph edge in Obsidian + clickable link in GitHub. The second is invisible to both.

### Pre-commit self-check

```bash
# From the wiki root, verify your new file has an inbound link
NEW=playbooks/<your-new>.md
grep -rln "$(basename "$NEW")" --include='*.md' . | grep -v "^./$NEW"
# Must print at least one path that isn't your new file
```

If grep returns nothing → add the link before pushing. A wiki commit that creates an orphan file gets reverted on review.

## 8. Commit messages

Format:

```
docs(<scope>): <subject>

Body — paragraphs explaining what changed + why. Reference commit hashes
in source repos when relevant. Reference IRON-RULES §N when codifying a
new rule.

Co-Authored-By: …  (auto)
```

Scopes:
- `docs(desks)` — anything touching the webapp desk model
- `docs(rules)` — IRON-RULES additions
- `docs(<project>)` — single-project-file edits, e.g. `docs(claudeflow)`
- `docs(playbook-<topic>)` — playbook additions
- `sync(<project>)` — drift catch-up; one project's wiki entry brought up to current HEAD
- `fix(<area>)` — corrections to typos / broken links / stale info

Subject ≤ 70 chars. Body wraps at 72.

### Commit message examples

**GOOD** — subject is concrete + body explains the why:

```
docs(rules): IRON-RULES §23 Engineering integrity (CEO 2026-04-29)

Three CEO-mandated rules + 2 Secretary suggestions, codified as §23.
Auto-injected via DEV-PROMPT-INLINE.md @-import so every agent sees
them at session start regardless of project.

§23.1 NO MAGIC — state assumptions, don't fabricate infra
§23.2 VERIFY BEFORE DONE — paste output, "should work now" forbidden
§23.3 DISSENT — surface blast radius before major change
```

```
sync(audit-2026-04-28): per-project wiki updates from weekly drift scan

mooniex-webapp:  HEAD 43caa80 (was 06d95ec) — 71 commits chronicled.
mooniex-claudeflow: v0.10.4 Lunar max_tokens 1024→4000 hard rule.
mooniex-option:  TRADE_DRY_RUN paper-data mode + Windows watcher fix.
```

```
fix(wiki): connect orphan claudeflow-verify-broker-handoff playbook

§7 audit found 1 file with 0 inbound links. Added inbound from
IRON-RULES §24 tool catalog + projects/mooniex-claudeflow.md
"Pending" section.
```

**BAD** — vague + no evidence:

```
fix: stuff
```
```
docs: update
```
```
fix(wiki): should work now
```

The third example violates IRON-RULES §23.2 — never use "should work now". Paste verification or describe the trace.

## 9. When to log to `MIGRATIONS.md`

Log a new top-of-file entry **only** for disruptive changes — anything that breaks an agent's existing memory if they don't read the diff:

- File / folder rename
- Section reorder in IRON-RULES (numbers shift)
- Reversal of a previously-stated rule
- Schema break in a documented DB table
- Rename of a recurring concept (e.g. "claim" → "lease")

**Don't** log additions: new IRON-RULES section, new project file, new playbook. A `git diff origin/main` covers those.

Entry format = headline (date + title), then four sections: "What changed", "Why", "How to reconcile your memory", "Watch-outs".

## 10. Pre-push checklist (every wiki commit)

```bash
cd /Users/gob/projects/LLMs

# 1. Pull first (other desks may have pushed)
git fetch origin && git pull --ff-only

# 2. Cross-link audit — broken relative links?
grep -rn '](\.\./' --include='*.md' . | grep -E '\.\./[^)]+\)' \
  | awk -F'](' '{print $2}' | awk -F')' '{print $1}' | sort -u | \
  while read link; do
    [ -e "$link" ] || echo "BROKEN: $link"
  done | head

# 3. Naming convention check (kebab-case + lowercase, except UPPERCASE top-level)
ls projects/ playbooks/ | grep -vE '^[a-z][a-z0-9-]+\.md$' | head

# 4. Stale folder references (after a rename — see MIGRATIONS.md)
# Tailor this grep to the rename you just shipped.

# 5. Commit + push
git add -A && git commit -m "docs(<scope>): …"
git push origin main
```

## 11. Wiki self-update workflow

If you change a wiki rule in this file, also:

1. Add a `MIGRATIONS.md` entry only if the change breaks how older agents wrote the wiki (rare).
2. Update `README.md` if you added a new top-level file.
3. Tell other agents in your commit body — they read the body when pulling.

## 12. Antipatterns (don't do)

- ❌ Paraphrasing a source file's hard rule. Always verbatim, blockquoted.
- ❌ Adding new top-level UPPERCASE files on a whim. Propose in MIGRATIONS.md first.
- ❌ Linking to absolute paths inside the wiki. Use relative links so the wiki ports cleanly.
- ❌ Inlining a 50-line code block when a link to the source file does the job.
- ❌ Multiple concerns in one commit. One scope, one commit.
- ❌ Writing the wiki in caveman / shorthand. Even when user is in caveman mode.
- ❌ "Note: this might be outdated" disclaimers in body. If outdated, fix it. If you can't, link to the source-of-truth file.
- ❌ Editing `MIGRATIONS.md` entries after they're shipped. Append a follow-up entry instead.

## 13. Per-section ownership

See [MAINTAINERS.md](MAINTAINERS.md). Short version:
- DEV who edits a project's source rule owns the matching wiki entry's update
- IRON-RULES §N is owned by whoever first introduced that rule
- INDEX / CROSS-FOLDER-MAP / MIGRATIONS / this file = "tribal" — last DEV to edit owns the next consistency pass

## 14. Incident postmortem template

When a production incident happens, log it under "Recent incidents" in the affected `projects/<name>.md`. Use this template — keep entries chronological, newest at top of that section.

```markdown
### YYYY-MM-DD HH:MM — <one-line title> (severity: P0/P1/P2/P3)

**Symptom**: what users / dashboards saw.

**Trigger**: the commit / config change / external event that started it.

**Blast radius**: who / how many were affected; how long.

**Root cause**: the actual broken assumption, not the symptom (per IRON-RULES §23.5).

**Fix**: commit hash + one-line description. Link to PR if any.

**Followups**: bullet list — wiki rule additions, monitoring gaps, refactors. Move to `MIGRATIONS.md` if the rule is new.

**Detection**: how we found out (Sentry alert / user report / cron failure / smoke). Note if detection was slow.
```

Severity scale (matches `IRON-RULES §6` Sentry alert tiers):
- P0 — money at stake (refund drift, payment failure), or webhook missing → revenue loss
- P1 — broken core flow (login, dashboard, broker sync)
- P2 — secondary feature degraded (course playback, image studio fail)
- P3 — cosmetic / single-user

If the postmortem produces a new rule, also append to `IRON-RULES.md` (or the relevant playbook) and log the addition in `MIGRATIONS.md` per §9.
