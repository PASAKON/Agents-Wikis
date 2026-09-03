# research/ — the org's research library

Everything the org spends real tokens finding out lives here, so nobody pays to
find it twice.

Opened 2026-09-03, after a day in which five research workflows ran and two of
them survived only as files in a temp directory that gets wiped. The Seedance
prompt-structure sweep alone was ten agents and about 1.1 million tokens.

## The two zones — this is the whole design

> "งานวิจัยต้องมีสิ่งที่เชื่อถือได้ไม่ว่าจะผ่านไปนานแค่ไหน กับอีกอย่างนึงคือ
> ต้องค้นหาเฉพาะจุดก่อนนำมาใช้งานจริง" — CEO, 2026-09-03

A research page is not one kind of thing. Half of it stays true forever and half
of it starts rotting the moment it is written, and mixing them is what turns a
library into a source of confident wrong answers. So every page is split:

### 🟢 DURABLE — true regardless of how much time passes

Write here: **what we measured ourselves**, **what was ruled out and how**, the
reasoning, the method, the counter-examples, the things that turned out to be
wrong and why.

This is the half that ages well. "S1C failed three generations while its previz
was 640x360, and five shots whose previz was 1280x720 all passed" is a fact
about our own history and is true in ten years. So is "we checked all 33 previz
files and every one is exactly 16:9, so aspect ratio is not the cause."

Negative results belong here and are worth as much as positive ones. Most of a
day's cost is spent eliminating things; if the elimination is not written down,
the next person eliminates them again.

### 🟡 PERISHABLE — verify the specific point before you act on it

Write here: **anything about somebody else's product.** Vendor behaviour, API
limits, prices, model versions, UI mechanics, what a platform's docs say today.

The rule for this zone is the CEO's second sentence: **spot-check the one claim
you are about to rely on, not the whole page.** If you are about to fire a
generation and the page says the platform accepts one video reference, check
that one line. Do not re-run the sweep. A sweep costs a million tokens; opening
one page costs nothing.

Each perishable claim carries the date it was verified. When you spot-check it,
update that date in place — that is the cheapest possible contribution to the
library and it keeps the page alive.

## Every claim states how it is known

- **official** — read at the vendor's own documentation
- **practitioner** — a named person reporting what they actually did
- **secondhand** — a site repeating it without a source
- **inference** — our own reasoning, not anybody's finding

This is not bookkeeping. Half the value of the Seedance sweep was discovering
that a "six-part formula" repeated across dozens of sites has no primary source
behind it at all. That is only visible if the level is recorded.

## Corrections are appended, never quietly overwritten

When a finding turns out to be wrong, add the correction with its date and its
evidence, and leave the original visible. On the day this library opened the CTO
was confidently wrong twice in one afternoon — about duplicate video references,
and about a file being "nine times heavier" when it ranked fifth of thirty-three
— and both corrections came from measuring rather than arguing. A page that
hides its own history teaches nobody how the answer was reached, and a reader
who cannot see a claim was once wrong cannot calibrate the rest of it.

## Naming and provenance

`YYYY-MM-DD-short-question.md`. The date is when the research ran and it is part
of the claim.

Head each page with when it ran, which session, the workflow run id, and the
method — how many agents, what was searched, whether findings were adversarially
re-checked. A reader six months from now needs to judge how much weight a claim
can carry.

## What belongs here

A page per question answered. Not per project, not per task. If someone might
ask the same question again in three months, it belongs here.

Not here: a decision (`decisions/`), a how-to for a recurring job
(`playbooks/`), the state of a project (`projects/`), or a task's deliverable
(that stays in the project repo).

## Filing

`scripts/research-file.py <run-id> --title ... --question ...` in the Agents
repo reads a workflow's own output and writes the page with its provenance
attached. It refuses to write a second page for a question already answered.

## Before opening a research task

Read this directory first — IRON §47. If the question is answered here, use the
answer. If it is answered but the perishable half is old, spot-check the
specific claim and update it in place. Never write a rival page.
