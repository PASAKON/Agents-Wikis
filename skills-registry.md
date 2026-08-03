# mooniex-claude-skills — Registry

Curated, merged Claude skills. Repo: **`PASAKON/mooniex-claude-skills`** (https://github.com/PASAKON/mooniex-claude-skills). **One skill per domain** → Claude picks unambiguously. Governance + freshness: `playbooks/skill-maintenance.md`.

## Naming
Skill `name:` (frontmatter + folder + install + wiki) = **lowercase-hyphenated** `mooniex-<domain>-skill` (Claude requires lowercase). Display title = Title Case for readability.

## Install
`npx skills add PASAKON/mooniex-claude-skills/<skill>` — OR symlink the repo clone into `~/.claude/skills/<skill>` (repo-backed: `git pull` = instant update, no re-install). Restart the session to load.

## Skills (✅ all shipped 2026-06-05)
| Skill | Domain | Sources merged | License |
|---|---|---|---|
| `mooniex-seo-skill` | SEO | claude-seo + ecc:seo + programmatic-seo | MIT (`b76459c`) |
| `mooniex-growth-skill` | Growth / marketing-ops (ICP, positioning, pricing, paid+Meta ads, CRO, A/B, competitor, GTM, analytics) | 8 MoonieX-original + paid-advertising (MIT/AityTech) + analytics (MIT/Corey Haines) | MIT + MoonieX (`071e96c`) |
| `mooniex-content-skill` | Content / copy (copywriting, editing, strategy, brand voice, channel, long-form) | MoonieX-original + ECC brand-voice/article-writing/content-engine (MIT) | MIT + MoonieX (`071e96c`) |
| `mooniex-tool-builder` | Build web tools + production UI | anthropics/skills web-artifacts-builder + frontend-design | **Apache-2.0** (`071e96c`) |

## ❌ EXCLUDED (proprietary — no permissive license; never shipped)
Brian Wagner skills: `content-idea-generator`, `voice-extractor`, `homepage-audit`, `positioning-basics`, `marketing-principles` (real dirs, commercial author byline / gumroad, no LICENSE). Only generic non-copyrightable concepts were reimplemented in original wording.

## Rules
- Prefer the mooniex skill over the upstreams it merged (CLAUDE.md prefer-rules added for SEO / growth / content / tool-builder).
- Per-feature credit + pinned versions in each `SOURCES.md`; upstream license texts in `NOTICE.md`.
- License: permissive only (MIT/Apache/BSD) or MoonieX-owned. Per-skill — `mooniex-tool-builder` is Apache-2.0, the rest MIT/MoonieX.

## Kept separate (NOT merged)
- `homepage-audit` is both excluded (license) AND a different domain (CRO) from SEO.
- `debug-mantra` / `post-mortem` / `scrutinize` (9arm) — prefer-ruled, not merged.
