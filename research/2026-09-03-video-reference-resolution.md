# A video reference below the output resolution fails the generation

**Question** why did S1C fail three generations while comparable shots passed
**Researched** 2026-09-03 · CTO session 116d7688 · measured, not searched
**Method** elimination across our own 33-file previz corpus and shot ledger, then a single controlled change

---

## 🟢 DURABLE — our own measurement, true regardless of what any vendor does later

### The result

S1C fired four times. Takes 1-3 used a 640x360 previz as the `@Video 1`
reference and all three died. Take 4 used the same previz scaled to 1280x720,
with nothing else changed, and passed:
`absence-S1C-take4-023e91c3-PASS-8s-720p.mp4`.

Same camera, same 8 seconds, same 192 frames, same 24fps, same Element
bindings, same spec fields. Resolution was the only variable.

### The failure signature

Each failure fired cleanly — a real "Generation started", a real job, a full
render of roughly 26 minutes — and then died on:

> "Something went wrong. Please try again, or change your input files or prompt."

All three were refunded automatically. Net cost across the whole investigation:
**$0**, because every fire was on higgsfield.ai under Unlimited.

### What was eliminated first, and how

This is the expensive half and the part worth keeping:

| Hypothesis | How it died |
|---|---|
| Aspect ratio mismatch | All 33 previz measured with ffprobe: every one exactly 16:9, S1C included |
| File too large / "heavy input file" | S1C is 696KB; S5-Render-v2 is 3,804KB and generates fine. S1C's raw bitrate ranks 5th of 33 — not an outlier |
| Duplicate `@Video` mention in the prompt | Counted file by file across all 16 prompt files. No duplicate chip anywhere in the corpus |
| A dead Generate button | Reproduced and fixed separately by restarting Chrome; takes 2-4 all fired cleanly |
| The prompt itself | A real contributing defect was found and removed, but take 4 passed carrying the same shot text |

### The correlation that survived

Of previz used as video references: **both at 640x360 failed, all five at
1280x720 passed.** Nine shots, no counter-example.

### Two claims of mine that were wrong, and why they were wrong

Recorded because the corrections were more useful than the original claims.

1. **"There must be a second `@Video` mention."** There was not. A count killed
   it. I had reasoned from a plausible mechanism instead of measuring first.
2. **"S1C's file is nine times heavier than the others."** It is not. I had
   compared it against S18a — one of the lightest files in the corpus — rather
   than against the distribution. Ranked properly it sits fifth of thirty-three.

Both died to measurement, neither to argument. That is the pattern worth
remembering.

---

## 🟡 PERISHABLE — verify the specific line before acting on it

**Higgsfield publishes no video-reference resolution requirement anywhere.**
*(verified 2026-09-03 · official · a sweep of all 74 help-centre articles, 20
docs pages, and the full 168KB OpenAPI spec found nothing on reference
resolution, minimum or relative.)*

So the rule above is **ours, not theirs.** It describes how their platform
behaved for our pipeline on this date, at 720p output. It is not a documented
contract and they are free to change it without telling anyone.

**What is published** *(verified 2026-09-03 · official)*:
- 3 video clips per generation
- up to 15 seconds each (stated on the Seedance 2.0 product page only)
- `video/mp4` only through the REST API

Note the first line contradicts our own `PREVIZ-INDEX.md`, which says exactly
one video reference. The composer may differ from the API. Unresolved.

### Before relying on this page

If you are about to fire and this matters, check the one thing you depend on:
that your previz is at least the output resolution. Do not re-run the
investigation. If we ever generate at 1080p, assume the threshold moves with the
output and measure once rather than trusting this page.

`scripts/previz-check.py` enforces the 1280x720 rule mechanically, along with
frame rate, duration, codec, a stray audio track, and bitrate band.
