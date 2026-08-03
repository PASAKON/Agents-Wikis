# Playbook — Skill Maintenance (mooniex-claude-skills freshness)

**Owner:** CTO. **Why:** our `mooniex-claude-skills` are curated forks that combine the *best features* of several upstream skills into ONE skill per domain — so Claude selects unambiguously (no "which skill?" problem). Upstreams keep updating separately → our forks can drift. This playbook keeps them fresh + legal.

## Principle
Merge **only the good features** of upstream skills into one `mooniex-<domain>-skill`. Each ships a `SOURCES.md` crediting every upstream + pinning the examined commit/version. We do **not** chase every upstream change — only port fixes/features that solve a real need.

## Repo layout
`PASAKON/mooniex-claude-skills` — one folder per skill:
```
mooniex-<domain>-skill/
  SKILL.md      (curated, merged)
  SOURCES.md    (per-feature credit + GitHub link + license + pinned version)
  references/  scripts/   (vetted, attributed)
```
Install one skill: `npx skills add PASAKON/mooniex-claude-skills/mooniex-<domain>-skill`

## Freshness ritual (CTO — monthly OR on-demand when a skill underperforms)
For each skill, open its `SOURCES.md` "pinned versions". For each upstream source:
1. Check the upstream repo for changes since the pinned commit/version (GitHub compare / releases).
2. Skim the diff. Ask: does it **fix a problem we hit** or add a feature we want?
3. **Yes** → port into our `SKILL.md` (keep our curation/dedupe), update `SOURCES.md` (new pinned commit + note what was ported), commit.
4. **No** (cosmetic / churn / irrelevant) → just bump the "last checked" date in `SOURCES.md`. Don't port.

## Rules (legal + sane)
- Every source credited in `SOURCES.md`: name + GitHub link + license. Keep upstream LICENSE/NOTICE.
- Combine **permissive licenses only** (MIT/Apache/BSD). Flag/exclude anything else.
- Copy only the good parts — keep each skill **small + focused** (easier to diff vs upstream).
- After a mooniex skill ships, **uninstall the upstream duplicates** it replaced → only the mooniex one stays in the index (this is the whole point: kills selection ambiguity).
- Never claim sole authorship.

## When NOT to fork-merge
- Upstream updates very frequently (high drift) → keep using it directly + a CLAUDE.md prefer-rule instead.
- A skill you barely use → not worth the maintenance.

## Registry
All mooniex skills tracked in `skills-registry.md`.
