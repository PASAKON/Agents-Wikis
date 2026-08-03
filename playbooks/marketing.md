---
title: "Marketing Playbook (CMO operating model)"
tags: [playbook, marketing, cmo]
status: Active
date: 2026-05-25
---

# Marketing Playbook

CMO operating model + team boundaries. CMO reads this on every
campaign brief. Workers under CMO read this on every task.

## CMO Team (direct reports)

| Role | File | Owns |
|------|------|------|
| `content_strategist` | `roles/content_strategist.md` | voice, messaging, copy, editorial calendar, SEO |
| `web_designer` | `roles/web_designer.md` | visual creative, landing pages, ad image/video |
| `ads_manager` | `roles/ads_manager.md` | paid media execution (launch, monitor, pause) |

Shared (not direct report — coordinate cross-functionally):
- `data_analyst` — performance reporting (primary owner: CGO)
- `developer` — Pixel / Conversions API / tracking integration (primary owner: CTO)

## Campaign Pipeline (canonical)

```
CEO brief
  └─ CMO
      ├─ wiki check (brand, IRON-RULES, past campaigns)
      ├─ check ad account state (page, pixel, budget)
      ├─ ping CFO if spend > per-campaign threshold
      ├─ ping CTO if new pixel / page / landing needed
      ├─ delegate parallel:
      │    ├─ content_strategist → copy variants (3-5)
      │    └─ web_designer → visual variants (3-5)
      ├─ review brand fidelity → pass or reopen (max 3 iter)
      ├─ delegate ads_manager → launch in test mode ($20/day x 3-5 days)
      ├─ handoff CGO → A/B + scale call
      └─ campaign retro ADR in decisions/
```

## Budget Authority

See `decisions/marketing-budget-authority.md` for per-channel caps.
TL;DR: any single campaign > $200 lifetime spend = CFO approval first.
Daily budget changes within an approved campaign = CMO discretion.

## Channel Mix (default)

| Stage | Channel | % of paid budget |
|---|---|---|
| Awareness | FB/IG video reach | 30% |
| Consideration | FB/IG carousel + traffic to landing | 40% |
| Conversion | FB/IG retargeting + LINE OA push | 30% |

Re-balance per campaign objective; document deviation in ADR.

## Brand Fidelity Gate

Every asset must pass:

1. **Voice check** — matches brand voice guide for that sub-brand
2. **Visual check** — uses approved color tokens + type system
3. **Claim check** — no product claim not backed by wiki / ADR
4. **Channel-fit check** — character limits + format ratios respected

Failed gate = `reopen_task` with specific line-item feedback. Max 3
iterations; on 3rd failure, escalate to CEO.

## Sub-brands

| Brand | Voice ADR | Notes |
|-------|-----------|-------|
| MoonieX | parent | `mooniex-voice-guide.md` is ground truth |
| WarpClip | WarpClip-design ADR-0006 v2.0 | B&W premium, single accent, em-dash ban |
| LungNote | LungNote-design ADRs (TBD) | clinical voice, evidence-led |
| Mooniex Option / Alphatrader / Moonx | parent voice | trader-direct, no fluff |

## Wiki Updates from CMO

- New campaign retro → `decisions/<YYYY-MM-DD>-<slug>-retro.md`
- New brand decision → propose ADR in relevant `*-wikis/40-Decisions/`
- New playbook tactic → update this file inline + commit
- New voice rule → update parent `mooniex-voice-guide.md` or sub-brand
  voice file (PR-style commit, CEO can review)

## Cross-CXO Coordination

| Decision | Coordinate with |
|----------|-----------------|
| New spend commitment | CFO |
| KPI hypothesis / A/B design | CGO |
| New web page / pixel / integration | CTO |
| User research / interview synthesis | CGO + CMO joint |
| Pricing copy / positioning vs price | CFO + CMO joint |

## Anti-patterns

- ❌ Launching ad without pre-registered success metric (CGO won't read it)
- ❌ Copy that contradicts wiki / brand ADR (auto-reject)
- ❌ Single creative variant (no A/B possible — always 3-5)
- ❌ Skipping the test phase to "save time" (cold launch at high spend)
- ❌ Approving spend without CFO sign-off above threshold
