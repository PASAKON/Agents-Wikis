# LLM Model Capabilities — org routing reference

> Lookup table: what each model the org can spawn is good at, its effort
> levels, and the real benchmark numbers behind the routing decisions.
> Source for `decisions/0009-model-routing-policy.md` + `decisions/0011-glm-5-2-zai-migration.md`
> and the spawn scripts. Updated 2026-08-01 (GLM-5.2 / Z.ai replaces GLM-5.1 / BytePlus).

## Quick pick

| Need | Pick | Why |
|---|---|---|
| Default daily coding / agentic | **Sonnet 5** | Best quality-per-quota; Claude Code's native xhigh target |
| Hardest arch / security / merge review | **Opus 4.8** | Frontier ceiling; +7 pts on hardest SWE-bench Pro |
| Bulk / offload / dodge Claude weekly cap | **GLM-5.2 (Z.ai)** | ~91–99% of Claude quality at ~1/6 cost; separate quota |
| Vision / screenshot / computer-use | **Opus/Sonnet only** | GLM-5.2 is text-only (0 on OSWorld) |

## Effort levels per provider

| Provider | Levels | Notes |
|---|---|---|
| Anthropic (Opus/Sonnet/Haiku/Fable) | `low` `medium` `high` `xhigh` `max` | `xhigh` = Claude Code native default for coding/agentic. `max` reserved for genuine escalation. |
| Z.ai GLM-5.2 | `high` `max` | Only 2 tiers. Z.ai recommends `max` for complex-task stability. **Endpoint rejects Claude `--effort`** → org spawn drops the flag on GLM path. |

## GLM-5.2 (Z.ai) — released 2026-06-16

- **Maker**: Z.ai (formerly Zhipu AI). 753B params, **MIT open weights** (Hugging Face).
- **Context**: 1M tokens opt-in via `glm-5.2[1m]` (set `CLAUDE_CODE_AUTO_COMPACT_WINDOW=1000000`). 131,072 max output.
- **Endpoint (Anthropic-compatible)**: `https://api.z.ai/api/anthropic` — drops into Claude Code with a base-URL swap. Coding Plan tiers Lite/Pro/Max/Team.
- **Pricing**: Coding Plan from ~$12.60/mo; API ~$0.56/M tok — roughly **1/6 of GPT-5.5** ([VentureBeat](https://venturebeat.com/technology/z-ais-open-weights-glm-5-2-beats-gpt-5-5-on-multiple-long-horizon-coding-benchmarks-for-1-6th-the-cost)).
- **Strengths**: long-horizon agentic tasks, tool-use (MCP-Atlas), math, deep research, code deployment, complex engineering. IndexShare attention cuts per-token FLOPs 2.9× at 1M context.
- **Weaknesses**: **text-only** (no image/computer-use → scores 0 on OSWorld-Verified); slower wall-clock (~3× Claude per Reddit reports); clear step behind Opus on the hardest long-horizon coding.

### Benchmarks — READ THIS CAVEAT FIRST

Cross-vendor numbers are **not** apples-to-apples: Z.ai and Anthropic measure on
different harnesses. There are exactly **two clean GLM-5.2-vs-Opus-4.8
comparisons** (Z.ai used Anthropic's own published Opus figure). **No
head-to-head GLM-vs-Sonnet data exists** — any such row is two harnesses stitched.
Analysis: [claudefa.st](https://claudefa.st/blog/models/glm-5-2-vs-opus-4-8-vs-sonnet-5).

**Clean (apples-to-apples, GLM-5.2 vs Opus 4.8):**

| Benchmark | GLM-5.2 | Opus 4.8 | Gap |
|---|---|---|---|
| SWE-bench Pro (agentic coding) | 62.1 | 69.2 | Opus +7.1 |
| HLE (with tools) | 54.7 | 57.9 | Opus +3.2 |

**Reference only (separate harnesses — read each column alone, NOT across):**

| Benchmark | GLM-5.2 (Z.ai) | Opus 4.8 (Anthropic) | Sonnet 5 (Anthropic) |
|---|---|---|---|
| SWE-bench Pro | 62.1 | 69.2 | 63.2 |
| Terminal-Bench 2.1 | 81.0 | 82.7 | 80.4 |
| HLE (with tools) | 54.7 | 57.9 | 57.4 |
| OSWorld-Verified (computer use) | none (text-only) | 83.4 | 81.2 |

> Trap: GLM-5.2's own best Terminal-Bench figure (82.7) shares the exact digits
> of Anthropic's official Opus 4.8 Terminal-Bench — different measurements on
> different harnesses that happen to coincide. **Not a tie.**

**GLM-5.2 vs GPT-5.5 (GLM wins each, per [VentureBeat](https://venturebeat.com/technology/z-ais-open-weights-glm-5-2-beats-gpt-5-5-on-multiple-long-horizon-coding-benchmarks-for-1-6th-the-cost)):**
SWE-bench Pro 62.1 vs 58.6 · FrontierSWE 74.4 vs 72.6 · MCP-Atlas 77.0 vs 75.3 ·
HLE 54.7 vs 52.2. Near-ties Opus 4.8 on FrontierSWE (74.4 vs 75.1) and MCP-Atlas (77.0 vs 77.8).

### Approx intelligence vs Claude (rough, for routing gut-feel only)

- **vs Sonnet 5 ≈ 97–99%** — within 1–2 pts on most coding benches, mixed ahead/behind (ahead on Terminal-Bench, behind on SWE-bench Pro).
- **vs Opus 4.8 ≈ 91–94%** — clean gap of 3–7 pts on the two fair comparisons.

These are *rough* — no clean GLM-vs-Sonnet head-to-head exists, so the Sonnet %
is stitched from two harnesses. Treat as "close to Sonnet, clear step below Opus."

## Claude Opus 4.8 / Sonnet 5 — the comparison anchors

Official Anthropic figures (these are what Z.ai used for the clean rows above):

| Benchmark | Opus 4.8 | Sonnet 5 |
|---|---|---|
| SWE-bench Pro | 69.2 | 63.2 |
| Terminal-Bench 2.1 | 82.7 | 80.4 |
| HLE (with tools) | 57.9 | 57.4 |
| OSWorld-Verified | 83.4 | 81.2 |

- **Opus 4.8**: frontier ceiling, multi-modal (vision + computer-use). ~2.5× Sonnet cost during Sonnet intro-pricing window (through 2026-08-31: Opus $5/$25, Sonnet $2/$10 per MTok).
- **Sonnet 5**: Anthropic frames as "near-Opus on coding/agentic." Best quality-per-quota; Claude Code's native xhigh default target.

## Org routing (post-2026-08-01)

BytePlus ModelArk removed; org is **Z.ai-only** for GLM offload. All worker DEV
roles + all four C-levels route to GLM-5.2 when the provider flag is set
(`DEV_MODEL_PROVIDER=zai` / `CXO_MODEL_PROVIDER=zai`), else Claude. See
`decisions/0011-glm-5-2-zai-migration.md`.

## Sources

- [Z.ai — GLM-5.2: Built for Long-Horizon Tasks](https://z.ai/blog/glm-5.2)
- [VentureBeat — GLM-5.2 beats GPT-5.5 on long-horizon coding for 1/6 cost](https://venturebeat.com/technology/z-ais-open-weights-glm-5-2-beats-gpt-5-5-on-multiple-long-horizon-coding-benchmarks-for-1-6th-the-cost)
- [DataCamp — GLM-5.2 features, setup, benchmarks](https://www.datacamp.com/blog/glm-5-2)
- [claudefa.st — GLM 5.2 vs Opus 4.8 vs Sonnet 5 (clean-comparison analysis)](https://claudefa.st/blog/models/glm-5-2-vs-opus-4-8-vs-sonnet-5)
- [DeepLearning.ai The Batch — GLM-5.2 agentic performance](https://www.deeplearning.ai/the-batch/top-agentic-performance-low-cost)
