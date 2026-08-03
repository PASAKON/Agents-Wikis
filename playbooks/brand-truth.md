---
title: "Playbook: Brand-Truth registry (read before producing any brand asset)"
tags: [playbook, brand, design, agents-must-read]
owner: CTO
last_updated: 2026-05-26
---

# Playbook — Brand-Truth registry

**Why this exists:** brand decisions made by the CEO have historically lived
only in one Claude session's private auto-memory (`reference_*_brand_truth.md`),
invisible to every other agent. Result: CMO/web_designer briefed off a stale
wiki ADR, shipped off-brand creative (WarpClip sage/fal.ai incident 2026-05-26,
[issue #7](https://github.com/PASAKON/mooniex-agents/issues/7)). This page is the
**org-level pointer** so any agent (CMO, web_designer, content_strategist, copywriter,
DEV) finds the canonical brand spec without grepping private memory.

## Hard rule

**Before producing ANY asset in a brand's name, read that brand's canonical
Brand-Truth doc (below) AND verify against live code.** Private auto-memory is
NOT authoritative for other agents — the committed wiki doc is. The running
code wins over any wiki text; update the doc before changing code.

## Canonical Brand-Truth docs (per product)

| Product | Canonical doc | One-line truth |
|---|---|---|
| **WarpClip** | `WarpClip Projects/wikis/10-Architecture/Brand-Truth.md` (P0) | DARK zinc-950 `#09090b` + **neon lime `#CCFF00`** accent + dark-W-on-lime-tile mark. Sage `#6B8E6F`, gradients, emerald, pastels = **banned**. ADR-0006 (B&W/sage) = deferred/aspirational only. |
| **LungNote** | (TBD — create `LungNote Projects/wikis/.../Brand-Truth.md`) | not yet documented |
| **MoonieX** | `mooniex-llms-wiki/playbooks/mooniex-voice-guide.md` (voice) + (visual TBD) | voice canonical; visual brand-truth TBD |

## AI image-gen rule for ad creative (CEO 2026-05-26)

- **fal.ai is banned for ad creative** (CFO 2026-05-26 — fal-sourced creative
  must not charge `warpclip-fb-sales-v1`).
- **gpt-image ("GPT Image Gen2") IS allowed for paid ad creative, but ONLY when
  gated by the deterministic brand-linter** (Approval Gate,
  `mooniex-webapp/src/lib/approval/brand-lint.ts` — palette/hex/logo/aspect
  hard-check; off-hex → auto-reject + prompt refine). See ADR
  [`decisions/2026-05-26-approval-gate.md`](../decisions/2026-05-26-approval-gate.md).
- **Brand-strict UI / logo / chrome assets:** no gen API at all — manual
  composition (HTML/CSS→Playwright, PIL, SVG), exact hex only.

## Process rule — no silent brand-truth updates

When the CEO updates a brand decision:
1. CTO mirrors it into the product's canonical Brand-Truth doc (wiki) — same day.
2. CTO/CMO re-brief every in-flight agent owning that brand's assets
   (`notify_cxo` / DM) — no relying on agents to re-read.
3. Any wiki ADR that now contradicts the Brand-Truth doc → mark `status: Deferred/Superseded`.

## See also
- [`decisions/2026-05-26-approval-gate.md`](../decisions/2026-05-26-approval-gate.md) — Approval Gate (brand-linter gate)
- [`playbooks/cfo-spend-approval.md`](https://github.com/PASAKON/MoonieX-Wikis/blob/main/playbooks/cfo-spend-approval.md) (`mooniex:playbooks/cfo-spend-approval.md`) — creative-asset spend must pass brand-truth gate before budget check
