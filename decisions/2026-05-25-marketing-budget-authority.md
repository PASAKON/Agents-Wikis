---
title: "ADR: Marketing budget authority + spend gates"
tags: [adr, marketing, cmo, cfo, budget]
status: Accepted
date: 2026-05-25
authors: [CMO, CFO, CEO]
---

# ADR — Marketing budget authority + spend gates

## Status

Accepted 2026-05-25.

## Context

CMO was spun up 2026-05-25 alongside CGO + CFO. Before any paid
media commitment, the org needs a written threshold for what CMO
can approve standalone vs what routes through CFO. WarpClip's first
paid FB campaign (CEO brief, same day) forced the decision.

CFO-Budget wiki for WarpClip ([wikis/50-Workflows/CFO-Budget.md])
already lists "launch ad ผ่าน Meta / TikTok / YouTube" as a
CFO-required gate. This ADR formalizes the threshold so CMO can
move quickly on small tests without queue-stalling.

## Decision

### Per-campaign spend authority

| Lifetime spend (single campaign) | Approval required |
|---|---|
| ≤ $50 | CMO standalone |
| $51–$200 | CMO sign-off + CFO notification (async, no block) |
| $201–$500 | CFO explicit approval before launch |
| $501–$2000 | CFO + CEO joint approval |
| > $2000 | CEO + board-equivalent (escalate) |

### Per-brand campaign cap (override default)

| Brand | Lifetime cap per campaign | Notes |
|---|---|---|
| WarpClip | $300 | CEO-approved 2026-05-25 for first FB Sales test |
| MoonieX (parent) | $500 | reserve for product launches |
| LungNote | $200 | early-stage, conserve |
| Mooniex Option / Alphatrader / Moonx | $300 | trader-direct audiences |

Daily budget changes within an already-approved campaign = CMO
discretion (so CMO can shift between ad sets to chase winners).

### Channel-level monthly cap (rollup)

| Channel | Monthly cap | Owner of cap |
|---|---|---|
| Meta (FB + IG) | $1500 | CMO; CFO renews monthly |
| Google (Search + YT) | $1000 | CMO; CFO renews monthly |
| TikTok | $500 | CMO; CFO renews monthly |
| LINE OA paid | $300 | CMO; CFO renews monthly |
| Influencer / one-off | per deal | CFO + CEO |

### Required artifacts before a paid launch

1. Campaign brief in `decisions/<date>-<slug>-brief.md` (CMO writes)
2. Success metric pre-registered by CGO (`primary_metric`, `min_sample`)
3. CFO line item if > $50 — wiki entry in `projects/finance.md`
   monthly close section
4. Pixel + Conversions API live (if Sales objective) OR documented
   Traffic-fallback decision

### Post-launch artifacts

1. Daily snapshot from `ads_manager` to wiki `projects/<brand>.md`
2. End-of-campaign retro ADR in `decisions/`
3. Cost reconciliation against `projects/finance.md` at month close

## Consequences

**Positive**
- CMO can run small ($≤50) creative tests without CFO blocking
- CFO has clear gate at $200 to catch budget drift early
- Per-brand caps prevent one product from starving others
- Channel monthly cap = predictable burn for runway math
- Daily-budget shift inside an approved campaign stays nimble

**Negative**
- 5-tier threshold = bookkeeping overhead (worth it for audit trail)
- $50 standalone authority is small — CMO may feel constrained on
  Thai-market USD-equivalent tests; revisit Q3
- Influencer / one-off deals require CFO+CEO every time (intentional;
  these are high-risk for brand)

## Alternatives Considered

- **Single global cap (e.g., CMO ≤ $500 always)** — simpler but
  doesn't reflect per-brand risk tolerance.
- **CFO approves every dollar** — too slow for creative iteration.
- **CMO unconstrained up to monthly channel cap** — too loose,
  no per-campaign rationalization.

## Rollback / Revisit

Revisit at end of Q3 2026 OR after WarpClip's first 3 campaigns
ship (whichever first). Adjust thresholds based on:
- Did the $50 standalone cap actually enable test velocity?
- Did the $200 CFO gate catch overruns or just slow CMO down?
- Did per-brand caps reflect actual ROAS distribution?

## See Also

- [WarpClip CFO-Budget workflow](https://github.com/PASAKON/WarpClip-wikis/blob/main/50-Workflows/CFO-Budget.md)
- `playbooks/marketing.md` (CMO operating model)
- `playbooks/ads.md` (paid media playbook)
- `roles/cmo.md`, `roles/cfo.md`, `roles/cgo.md`, `roles/ads_manager.md`,
  `roles/content_strategist.md`
