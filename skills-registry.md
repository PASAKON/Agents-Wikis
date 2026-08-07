# Skills — Registry

Where every org-owned skill lives, what it is allowed to cost, and how to name a
new one. Governance + freshness ritual: `playbooks/skill-maintenance.md`.
Rationale for the current shape: [ADR 0015](decisions/0015-skill-context-budget.md).

## The cost rule (read this before adding anything)

Only `name` and `description` are injected into **every** session. Everything
else in a SKILL.md — `owner`, `origin`, `scope`, and the whole body — is read
only when the skill is actually invoked.

- `description` is scarce. Target **≤ 350 chars**. English prose; keep Thai
  trigger phrases verbatim, because they are matcher data, not prose.
- `owner` / `origin` / `scope` are free. Always fill them in.
- The body is free. Make it as thorough as the task needs.

`skillOverrides: "off"` in `~/.claude/settings.json` does **not** reduce cost —
it suppresses auto-invocation while the entry still loads. To actually remove a
skill, move it out of the skills directory.

## Where skills live

| Scope | Path | For |
|---|---|---|
| **Private, org-internal** | `mooniex-agents/.claude/skills/` | Anything naming internal systems, money, cards, credentials, or CEO personal data |
| **Public, reusable** | `PASAKON/MoonieX-ClaudeSkills` (symlinked into `~/.claude/skills/`) | Playbooks useful to anyone outside the org |
| **Curated external** | `~/.claude/skills/<name>` → symlink into `Projects/external/…` | Imported from an upstream we do not own |

**Public/private is a judgement made per skill, not per name.** `mooniex-finance`
carries card last-four digits mapped to vendors, a receipt number, and per-brand
budget caps — it lives in the private repo despite the `mooniex-` prefix.
`seedance-scene-prompt` is generically useful and internally clean, so it is
public despite having no prefix. Scan for credentials, PII, financial figures,
and internal paths **before** any move into the public repo: a push cannot be
taken back.

## Naming

- **`mooniex-<domain>`** when the domain word is generic enough to collide with
  another plugin's skill: `seo`, `growth`, `content`, `kpi`, `retention`,
  `finance`, `image-gen`.
- **No prefix** when the name is already unambiguous — `seedance-scene-prompt`
  names a specific model and output format, so `mooniex-` would add characters
  without adding meaning.
- **No `-skill` suffix.** Everything here is a skill; it carried no information.
  (Retired 2026-08-07, `MoonieX-ClaudeSkills@a318eb7`.)
- Family prefixes are good: `session-open` / `-close` / `-list` / `-merge` /
  `-save` / `-worktree` — one word recalls the whole set.
- Verb-first for actions, noun for reference material. Lowercase-hyphenated.
  No version numbers in names; edit in place.

## Public — `PASAKON/MoonieX-ClaudeSkills`

Install: `npx skills add PASAKON/mooniex-claude-skills/<skill>` — or symlink the
repo clone into `~/.claude/skills/<skill>`, which makes `git pull` an instant
update with no re-install.

| Skill | Domain | Owner | License |
|---|---|---|---|
| `mooniex-seo` | Technical + on-page SEO, schema, CWV, content/E-E-A-T, programmatic, GEO | CGO | MIT |
| `mooniex-growth` | Acquisition + conversion — paid/Meta ads, CRO, A/B, funnels, ICP, positioning, pricing, GTM | CGO | MIT + MoonieX |
| `mooniex-content` | Copywriting, copy editing, content strategy, brand voice | CMO | MIT + MoonieX |
| `mooniex-kpi` | Measurement — KPI choice, attribution, CAC/LTV/ROAS, funnel + cohort, tracking plans | CGO | MIT + MoonieX |
| `mooniex-retention` | Activation, onboarding, habit loops, lifecycle email, churn, win-back, dunning | CGO | MIT + MoonieX |
| `mooniex-image-gen` | Image-prompt ideation via MeiGen MCP free tools. Does **not** generate — the prompt goes to Fal.ai | CMO | MoonieX |
| `mooniex-tool-builder` | Build web tools + production UI design | CTO | Apache-2.0 |
| `seedance-scene-prompt` | Cinematic video-gen scene prompts, REFERENCE + VISUAL + (DIALOGUE) + AUDIO/SFX | CMO | MoonieX |

Per-feature credit and pinned upstream versions live in each `SOURCES.md`;
upstream licence texts in `NOTICE.md`. Permissive licences only (MIT / Apache /
BSD) or MoonieX-owned.

## Private — `mooniex-agents/.claude/skills/`

| Skill | Purpose | Owner |
|---|---|---|
| `session-open` / `-close` / `-list` / `-merge` / `-save` / `-worktree` | Session lifecycle — charter, exit gate, inventory, carry-forward, state save, progress tree | CTO |
| `session-change-model` | Propose a mid-session tier escalation per ADR 0009 | CTO |
| `cto-merge-checklist` | Pre-merge gate; refuses `merge_task` if a check fails | CTO |
| `dev-spawn-protocol` | Required steps on every DEV spawn (IRON §29) | CTO |
| `skill-author` | Scaffold a new SKILL.md whose description reliably fires | CTO |
| `spawn-web-designer` | Boot the open-design surface for hands-on CEO design | CTO |
| `gdrive-filing` | Filing rules for the CEO's personal Google Drive | CFO |
| `mooniex-finance` | Org finance — budget, runway, subscriptions, `webapp_cfo_*`, card/vendor mapping | CFO |

## Excluded — proprietary, never shipped

Brian Wagner skills: `content-idea-generator`, `voice-extractor`,
`homepage-audit`, `positioning-basics`, `marketing-principles`. Commercial author
byline, no permissive licence. They stay as local real directories and must not
be merged, published, or removed as "duplicates" — only generic,
non-copyrightable concepts were ever reimplemented in original wording.

## Kept separate on purpose

- `homepage-audit` — CRO, a different domain from SEO (and excluded above).
- `debug-mantra` / `post-mortem` / `scrutinize` (9arm) — routed to by preference
  in `~/.claude/CLAUDE.md`, deliberately not merged.
