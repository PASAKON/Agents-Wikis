# ADR 2026-06-13 — Mockup before any paid image-gen (MoonieX creative)

**Status:** accepted · **Owner:** CMO · **Directive:** CEO 2026-06-13

## Context

Producing the Musk "$1-trillion / SpaceX IPO" newsjack poster, the CMO ran
paid `gpt-image-2` gens before a layout + copy were agreed. Output was wasted:
AI-rendered Thai came out garbled and the headline copy was wrong. CEO:
"การ Generate ภาพ ต้องสร้าง mockup ก่อน gen … Mock ช่วยลดเหตุการณ์นั้นได้."

## Decision (rule)

Before **any** credit-consuming image generation (fal.ai / gpt-image-2 / etc.)
for a MoonieX (or any brand) creative:

1. **Mockup first ($0).** Render a wireframe of the layout + exact copy with
   HTML→Playwright (or PIL): DASHED boxes = regions the AI will generate
   (hero photo, rocket, b-roll); SOLID styled text = the real overlay copy in
   final treatment (locked hex, Prompt font for Thai).
2. **CEO approves / adjusts** the mockup (position, size, wording) — no gen
   until approved.
3. **Gen SCENE ONLY.** The AI prompt makes the photographic scene with **no
   text, letters, numbers, or logo** — AI garbles text, Thai especially.
   Reserve dark negative space for the text layer.
4. **Overlay all copy crisp** (Playwright/PIL) on top of the AI scene — Thai
   never garbles this way; logo is composited, never AI-drawn.

## Why

A free mockup catches copy/layout/text errors with zero spend; the
scene-only + overlay split removes the single biggest defect source (baked
text). Complements the "state $ + wait for OK before paid API" rule.

## Applies to

All MoonieX creative image generation. Pipeline reference:
`Agents/scripts/spcx_musk_gen.py` (gen scene → compose overlay) and
`spcx_layout_mock.py` (wireframe). Recipe: memory `reference_mooniex_poster_recipe`.
