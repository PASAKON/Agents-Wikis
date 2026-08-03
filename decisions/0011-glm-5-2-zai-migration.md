# ADR 0011 — GLM-5.2 on Z.ai supersedes GLM-5.1 on BytePlus

- **Status**: Accepted — Implemented
- **Date**: 2026-08-01
- **Decider**: CEO (ตอนนี้มาใช้ Z.ai แล้ว, 2026-08-01)
- **Implemented by**: CTO session
- **Supersedes**: `decisions/0009-model-routing-policy.md` Tier 3 (GLM-5.1 via BytePlus)

## Context

ADR 0009 (2026-07-01) wired GLM offload to **GLM-5.1 via BytePlus ModelArk**
($10/mo, `/api/coding` endpoint). Two things changed:

1. **GLM-5.2 shipped** (2026-06-16, Z.ai). On the two clean apples-to-apples
   comparisons vs Opus 4.8 it closes most of the gap GLM-5.1 had —
   SWE-bench Pro 62.1 (was 58.4), and it beats GPT-5.5 on every long-horizon
   coding bench VentureBeat measured. Full numbers in
   `reference/llm-model-capabilities.md`.
2. **BytePlus Coding Plan quota proved unworkable** — exhausted mid-task
   (2026-07-02), reset window opaque, no console/API to read remaining quota
   (had to discover exhaustion from a 429 body). Z.ai Coding Plan has a real
   quota dashboard and an Anthropic-compatible endpoint (`/api/anthropic`).

CEO decision 2026-08-01: GLM-5.2 is "เก่งพอใช้ Dev ได้เลย" (no separate test
needed), org moves to Z.ai, BytePlus removed entirely. This **reverses** the
07-17 stance (`feedback_glm_small_tasks_only` — "route only small/simple work
to GLM") for the 5.2 generation.

## Decision

- **Provider**: Z.ai only. `https://api.z.ai/api/anthropic`.
- **Model**: `glm-5.2` (default). `glm-5.2[1m]` for 1M-context tasks.
- **BytePlus ModelArk removed** from `lib/config.py` provider registry,
  all spawn scripts, `.env`. `BYTEPLUS_API_KEY` line deleted (old key to be
  revoked by CEO on the BytePlus console).
- **Scope widened**: `DEV_MODEL_PROVIDER` now defaults to all worker DEV roles
  (developer, tester, web_designer, data_analyst, prompt_engineer,
  ads_manager, content_strategist) and `CXO_MODEL_PROVIDER` to all four
  C-levels (cto, cmo, cgo, cfo) — not just the 0009 pilot scope.
- **Effort**: GLM endpoint still rejects Claude `--effort`; org spawn drops
  the flag on the GLM path (GLM has its own `high`/`max` thinking modes).

## Routing still follows ADR 0009's tier logic

Claude (Sonnet 5 default, Opus 4.8 escalation) remains the quality ceiling for
security / arch / merge review. GLM-5.2 is the **bulk offload** tier — close
to Sonnet (~97–99%), clear step below Opus (~91–94%), at ~1/6 the cost and a
fully separate quota. Pick per `reference/llm-model-capabilities.md`.

## Follow-ups

- Revoke BytePlus API key on BytePlus console (CEO) — key already removed from repo.
- Cancel BytePlus Coding Plan subscription (CEO) — tracked in LungNote.
- Watch GLM-5.2 wall-clock (reported ~3× slower than Claude) on real DEV tasks.
- After ≥3 days stable on Z.ai, this ADR's migration is considered settled.
