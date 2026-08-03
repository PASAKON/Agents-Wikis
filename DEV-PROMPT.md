# DEV Prompt — paste into every project's CLAUDE.md / AGENTS.md

> **DEV = Claude / Codex agent operating inside a Mooniex project folder.**

Copy the block below into the bottom of `CLAUDE.md` (or `AGENTS.md`) in each `mooniex-*` project. This makes every DEV aware of the central wiki and their maintenance responsibility.

---

## Block to paste (verbatim)

```markdown
# Wiki sync (mandatory)

A central knowledge wiki lives at `/Users/gob/projects/LLMs/`. It documents this project's rules, deploy procedure, env handling, and cross-folder dependencies for every other Mooniex agent to read.

## Read

Before starting work that touches more than one folder, or before a `vercel deploy` / `git push`:
- `/Users/gob/projects/LLMs/INDEX.md` — project matrix
- `/Users/gob/projects/LLMs/projects/<this-folder>.md` — your project's facts
- `/Users/gob/projects/LLMs/IRON-RULES.md` — canonical rules (verbatim)
- `/Users/gob/projects/LLMs/playbooks/before-cross-folder-edit.md` if editing another folder

## Update (your responsibility)

You own the wiki entry for the project you're editing. Update wiki **before** `git push`, in the same change set.

| You changed | Update |
|-------------|--------|
| This project's `CLAUDE.md` / `AGENTS.md` / `setup-windows.md` | `LLMs/projects/<this>.md` + relevant section in `LLMs/IRON-RULES.md` |
| New webhook / env share / table FK to another project | `LLMs/CROSS-FOLDER-MAP.md` (mermaid edge + edge-detail section) |
| New folder under `/Users/gob/projects/` | Create `LLMs/projects/<new>.md` + row in `LLMs/INDEX.md` |
| Postmortem produced a new rule | Add to `LLMs/IRON-RULES.md` + "Recent incidents" in project file |

Do not paraphrase verbatim quotes — re-pull from source on each update.

Full ownership matrix: `LLMs/MAINTAINERS.md`.

## Skip wiki update for

- In-progress task state, today's TODOs
- Pure code changes that don't change a documented rule
- Bug fixes that don't change the rule itself
```

---

## How to install in each project

For each project folder:

```bash
cd /Users/gob/projects/<folder>

# Append the block to CLAUDE.md (or AGENTS.md, whichever the project uses)
cat >> CLAUDE.md <<'EOF'

# Wiki sync (mandatory)
... (paste block above)
EOF

# Commit + push
git add CLAUDE.md
git commit -m "docs: add LLMs wiki sync rule"
git push origin <branch>
```

## Recommended install order

Priority by edit-rights + visibility:

1. **mooniex-webapp** (and MXN — same `AGENTS.md`) — highest traffic, both agents will see it
2. **mooniex-claudeflow** — Hot Zone discipline already exists, fits the pattern
3. **mooniex-option** — `setup-windows.md` is the entry doc, append there
4. **mooniex-controller-system** — has handoff with claudeflow, needs to know
5. Others — lower priority

## Centralized alternative

Instead of pasting into every project, you can add a single line at the top of each project's `CLAUDE.md`:

```markdown
@/Users/gob/projects/LLMs/DEV-PROMPT-INLINE.md
```

(The `@` import works in Claude Code's CLAUDE.md hierarchy.) Then maintain the rules in **one** file. Trade-off: agents that don't auto-resolve `@` imports won't see it.

Recommended: paste verbatim into top 3 projects (webapp, claudeflow, option) for visibility; use `@` import for the rest.
