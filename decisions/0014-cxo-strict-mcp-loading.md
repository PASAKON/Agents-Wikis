# ADR 0014 — CXO sessions load a per-role MCP set under --strict-mcp-config

- **Date:** 2026-08-06 (amended 2026-08-07, see below)
- **Author:** CTO (session #4911053b), at CEO request
- **Status:** Accepted, shipped `cfa85bc` (mooniex-agents main); amended `d0d3229`
- **Supersedes:** nothing. Related: [0009 model routing](0009-model-routing-policy.md),
  [2026-07-12 org runtime to Contabo](2026-07-12-org-runtime-to-contabo.md)

## Context

The CEO reported the Mac was slow. Measurement, not guesswork, on the dev
box (MacBook Pro M1 2020, **8 GB RAM**, macOS 14.3.1):

| metric | measured |
|---|---|
| RAM unused | 143 MB |
| compressor | 2,142 MB |
| swapouts | 13,688 |
| 2 live claude sessions + MCP subtrees | **2,830 MB of 8,192** |

RSS is useless here — it fell from 1,873 MB to 568 MB in ten minutes as
macOS compressed idle pages. All figures above use `footprint -p <pid>`
(`phys_footprint`), which counts compressed pages.

Cost centre was MCP sprawl in C-level sessions: **13 servers, 1,489 MB**
for one CTO session whose `--allowed-tools` list only covers `mcp__org__*`
and `mcp__lungnote__*`. Each npx-based server costs twice — an `npm exec`
wrapper (~46 MB) holding the real node server (~43 MB).

Two compounding causes:

1. **No `--strict-mcp-config` on either CXO launcher.** `runners/dev_init.py`
   has passed it since it was written, so DEV spawns were never the problem
   — only the C-level ones.
2. **`enabledMcpjsonServers` declared in two files** — `~/.claude.json` and
   `.claude/settings.local.json`. Overlapping names spawn twice. Predicted
   6/6 against the live process list: four supabase refs (in both files) ran
   x2; `mooniex-coord` and `meta-ads-135` (one file) ran x1. Claude Code
   rewrites `settings.local.json` on its own, so this cannot be fixed by
   editing that file alone.

## Evidence

Usage audit over **199 transcripts / 6,478 logged MCP calls**
(2026-04-23..2026-08-06); 194 of 199 fall in the last 30 days, so all-time
equals current behaviour:

| MCP | calls | note |
|---|---:|---|
| claude-in-chrome | 4,918 | built-in, no process |
| org | 852 | core |
| lungnote | 285 | core |
| Drive / Gmail / FB Ads | 153 / 74 / 50 | remote connectors, no process |
| meigen | 43 | |
| ecc github | 39 | duplicate of `gh` CLI |
| supabase | 28 | |
| meta-ads-135 | 14 | |
| ecc exa | 5 | HTTP transport, no process |
| **ecc playwright** | **4** | duplicate of claude-in-chrome (4,918) |
| mooniex-coord | 2 | |
| **ecc context7 / memory / sequential-thinking** | **0** | |
| **polymarket** | **0** | arb project concluded |

## Decision

1. **One generator, `scripts/lib/cxo_mcp_config.py`**, shared by
   `cto-claude.sh` and `cxo-claude.sh`. Retires the drift between an inline
   heredoc and the committed `config/cto.mcp.json`, whose absolute Mac paths
   break on Contabo (`ROOT=/opt/mooniex-agents`).
2. **Per-role sets** — base `org, lungnote, mooniex-coord`; `+supabase` for
   cto/cfo/cgo; `+meta-ads-135` for cmo/cgo; `+meigen` for cmo.
   *(Amended 2026-08-07: `mooniex-coord` left the base — see below.)*
3. **`--strict-mcp-config` on both launchers**, default on.
4. **Skip servers that are not installed** rather than spawning one that
   fails its handshake — this is what makes the launcher portable.
5. **Prune plugin MCPs with zero or duplicated usage** — dropped context7,
   memory, sequential-thinking, playwright from the active ECC install at
   `~/.claude/plugins/cache/ecc/ecc/2.0.0-rc.1/.mcp.json`. Kept `github`
   (used) and `exa` (HTTP, free).
6. **`spawn-cto.sh` warns** when another CTO chat is live and points at
   `--last`. Advisory only — it never kills. A quiet chat is usually waiting
   on a DEV, and reaping on idleness alone has silently killed live work.

## Consequences

- CTO: **13 servers → 4**, ~1,489 MB → ~290 MB per session.
- ECC prune: ~356 MB off **every** session, including plain CEO ones.
- CXO loses the `github` MCP under strict mode. `gh` is already the
  documented org path and `Bash` is in ALLOWED, so this consolidates on one
  GitHub route instead of two.
- Escape hatches, no code edit: `CXO_STRICT_MCP=0` restores the old
  inherit-everything behaviour; `CXO_EXTRA_MCP` / `CXO_SKIP_MCP` adjust a
  role's set; `CXO_NO_SESSION_WARN=1` silences the spawn warning.
- A plugin version bump installs to a new `cache/.../<version>/` directory,
  so the ECC prune does not carry forward — re-apply after upgrading ECC.

## Verification

`test_spawn_tab_routing` 28 PASS · `test_model_routing_consistency` ALL PASS
· `bash -n` clean on all three launchers · generator emits valid JSON for
all four roles with the Supabase PAT left as a `${VAR}` placeholder.

Baseline for the next measurement is in the commit body: 2,830 MB across two
sessions, 143 MB unused, compressor 2,142 MB.

## Not addressed

8 GB soldered RAM is the real ceiling; this buys headroom, it does not lift
it. The structural fix stays [org runtime to
Contabo](2026-07-12-org-runtime-to-contabo.md) — Phase A done, B–F pending.

---

## Amendment — 2026-08-07 (CTO session #0934c3ae, CEO request)

Shipped `d0d3229`. Terminology first, because the original framing invited a
misreading: the split is keyed on **role**, never on **model**. Model choice
lives in `cto-claude.sh` `MODEL_ARGS` and `config/agents.yaml` (ADR 0009) and
has no relationship to which MCP servers load.

### Three gaps found auditing the shipped implementation

**G1 — a role loaded servers whose tools were not on its allowlist.** Both
launchers carried a hand-copied `ALLOWED` string naming `org` + `lungnote`
only, while role `cto` also loaded `supabase`. Worst of both worlds: the
server paid its process and schema cost on every launch, then went through a
permission prompt on every call. Both launchers now derive `--allowed-tools`
from the generator's new `--print-allowed`, which emits tools **only for
servers it actually emitted**, and takes `org`'s entries from
`lib/org_tools_registry` — the same single-source guarantee
`test_tool_parity.py` gives the three Python surfaces, now extended to the
two shell ones. Fixing G1 immediately exposed the identical bug in
`meta-ads-135`, which contributed zero allowlist entries.

**G2 — `mooniex-coord` was in the base set for all four roles** on the
strength of 2 calls out of 6,478. Demoted to opt-in (`CXO_EXTRA_MCP`).

**G3 — `CXO_EXTRA_MCP` / `CXO_SKIP_MCP` did nothing through the spawn
scripts.** `osascript`'s `write text` starts a fresh login shell, so the
caller's exports were gone before the launcher read them; the documented
overrides only ever worked when `cto-claude.sh` was invoked directly. Both
spawners now re-export the `CXO_*` set inside `CHAT_CMD`.

### Allowlist policy this establishes

Pre-approval tracks blast radius, not convenience:

- **supabase now runs `--read-only`** (opt out with `CXO_SUPABASE_WRITE=1`).
  That is the *only* reason `execute_sql` is safe to pre-approve — the
  guarantee is enforced by the server, not by prompt fatigue. Verified: a
  `CREATE TABLE` comes back `25006: cannot execute CREATE TABLE in a
  read-only transaction`. `apply_migration` stays off the list; prod DDL goes
  through the Supabase SQL Editor / db push, never a chat.
- **meta-ads-135 exposes 134 tools**, 68 of them read-shaped. Pre-approving
  all of them would just relocate the bloat, so 9 read tools are listed
  (accounts, campaigns, adsets, ads, and their insights). Every
  create/update/delete stays gated — those spend budget.
- **meigen's `generate_image` / `generate_video` are deliberately excluded.**
  They cost money, and MoonieX generates through Fal.ai, never MeiGen
  (CEO 2026-07-13). MeiGen is here for the free prompt/reference tools.

### Borrowing a server without relaunching

A strict session's MCP set is frozen at spawn. `claude mcp add` writes to a
config file strict mode ignores, so there is no mid-session attach. New
`scripts/mcp-borrow.sh` runs a throwaway headless `claude -p` holding exactly
one server, prints to stdout, and dies — the server is its child, so it dies
too. The calling session keeps its conversation and never pays for the
server. Stripped for cost: `--tools ""`, `--setting-sources ""` (no hooks, no
plugins, no skill catalog, no CLAUDE.md), a two-line replacement
`--system-prompt`, `--no-chrome` (the Chrome connector is built-in and
survives `--strict-mcp-config`), `--no-session-persistence`.

**`--raw` is the mode to reach for, and the reason is correctness, not just
cost.** It speaks MCP over stdio via `scripts/lib/mcp_call.py` with no model
in the loop. Benching *"how many open todos"* through a model returned **50,
53, 7 and 193** across two models and two runs. The answer is **193**; a raw
call returns it in ~1s for **zero tokens**. Two lessons: haiku cannot be
trusted to aggregate over a payload (7 was its answer with the limit pinned),
and an unpinned tool default silently truncates — `list_todos` caps at 50, so
the plausible-looking "50" was the cap, not the count. Use a model for
judgment (pick the best reference, summarise a thread), never for a number a
tool already knows.

### Residual, not addressed

- `meta-ads-135`'s 134 tool schemas still load in full for cgo/cmo sessions;
  only the allowlist was trimmed. Cutting the schema cost needs server-side
  tool filtering, if that server supports it.
- `cancel_todo` / `delete_todo` remain off lungnote's allowlist, matching the
  pre-existing 8-verb list. The CEO has granted standing permission to
  add/delete todos, so promoting `delete_todo` is a one-line change awaiting
  a deliberate call.
- 8 GB RAM is still the ceiling; 6 CTO + 1 CMO sessions were live during this
  audit, each with its own server subtree. Per-session duplication, not
  per-role over-inclusion, is now the dominant cost.

### Verification

`scripts/test_mcp_role_config.py` (new) 0 failures — it pins all three gaps,
the read-only default, registry parity for the shell surfaces, and executes
the spawn scripts' real `ENV_PREFIX` loop text rather than a paraphrase.
`test_tool_parity`, `test_model_routing_consistency`, `test_spawn_tab_routing`,
`test_osc_surface_boundary`, `test_iterm_typewriter` all pass. A real `claude`
run with the launcher's assembled argv called `mcp__org__list_projects` and
returned 11. Bare-login-shell check derives all 33 tools; a broken venv exits
1 and refuses to launch rather than silently emitting a short list.
