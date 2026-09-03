# research/ — the org's research library

Everything the org spends real tokens finding out lives here, so nobody pays to
find it twice.

Opened 2026-09-03, after a day in which five research workflows ran and two of
them survived only as files in a temp directory that gets wiped. The Seedance
prompt-structure sweep alone was ten agents and about 1.1 million tokens, and
its findings — including the official ByteDance statement that Seedance 2.0
ignores timestamps while 2.5 reads them — existed nowhere but `/tmp`.

## What belongs here

A page per question answered. Not a page per project, not a page per task.
If someone might ask the same question again in three months, it belongs here.

Suitable: how a vendor's API actually behaves, what a community does and where
the consensus is thin, why an approach was rejected, what a sweep of our own
corpus turned up, benchmark numbers we measured ourselves.

Not suitable: a decision (that is `decisions/`), a how-to for a recurring job
(that is `playbooks/`), the state of a project (that is `projects/`), or the
deliverable a task produced (that stays in the project repo).

## Naming

`YYYY-MM-DD-short-question.md` — the date is when the research ran, and it
matters, because a finding about a vendor's product can go stale without
anything in the file changing.

## Every page carries its own provenance

Head each page with when it ran, which session, the workflow run id, and the
method — how many agents, what was searched, whether findings were
adversarially re-checked. A reader six months from now needs to judge how much
weight a claim can carry.

## Mark the evidence level of every claim

The habit that makes this library worth having: say how you know.

- **official** — read at the vendor's own documentation
- **practitioner** — a named person reporting what they actually did
- **secondhand** — a tutorial site repeating something without a source
- **inference** — our own reasoning, not anybody's finding

Half the value of the Seedance sweep was discovering that a "six-part formula"
repeated across dozens of sites has no primary source behind it at all. That is
only visible if the level is recorded.

## Before opening a research task

Read this directory first. If the question is answered here, use the answer.
If it is answered here but the page is old enough to doubt, re-run the parts
that could have changed and update the page in place rather than writing a
second one.
