# Agents Wikis — the org runtime wiki

**How the agent org works.** Any agent, any project, any machine.

Created 2026-08-03 per ADR-0013 "Wiki scope boundary" (the ADR itself lives in
`MoonieX-Wikis/decisions/0013-wiki-scope-boundary.md` until Phase 4 moves it
here).

Namespace: **`org:`** — reach a page with `wiki_read("org:playbooks/x.md")`.
Roots are registered in `mooniex-agents/config/wikis.yaml`.

---

## What belongs here

> A page belongs to **this** wiki if it describes **how agents work** — any
> agent, any project.
>
> A page belongs to a **project wiki** if it describes **what that project is,
> or how to operate that specific codebase**.
>
> When a page contains both, it is **split**, not duplicated. Neither wiki
> restates the other; they cross-reference.

| Here (`org:`) | Not here |
|---|---|
| Spawn protocol, tab lifecycle, task queue, path locks | A project's stack, deploy target, DB prefix |
| Role definitions, permission policy, model routing | A project's hot zones or migration runner |
| Session discipline, merge authority, engineering integrity | A brand's voice, content strategy, campaigns |
| Language and writing rules that bind every agent | A project's incident log or changelog |

The org runs multiple brands on one runtime — MoonieX, LungNote, WarpClip,
LinkReed. Anything written here must hold for **all** of them. If a rule only
makes sense for one brand, it belongs in that brand's wiki.

## Project wikis

| Wiki | Repo | Scope |
|---|---|---|
| MoonieX | `PASAKON/MoonieX-Wikis` | MoonieX product, brand, finance |
| LungNote | `PASAKON/LungNote-Wikis` | LungNote only |
| WarpClip | `PASAKON/WarpClip-Wikis` | WarpClip only |
| LinkReed | `PASAKON/LinkReed-Wikis` | LinkReed only |

## Layout

```
decisions/   ADRs about the runtime itself (spawn, routing, sessions, tooling)
playbooks/   Step-by-step for operating the org (orchestration, tab lifecycle, …)
policies/    Standing policy (permissions, spend authority)
roles/       What each role is accountable for
```

## Precedence

This wiki is the **higher authority**. A project wiki may add rules; it may not
contradict what is written here. A project wiki that needs an org rule **links**
to it — copying is what produced the four-places-one-fact problem this split
exists to fix.

## Status

**Phase 2 of ADR-0013 — the repo exists and is wired; content has not moved
yet.** Until Phase 4 lands, the org rules still live in `MoonieX-Wikis`
(`IRON-RULES.md` and the playbooks named in the ADR). Read them there.
Do not start a second copy here.
