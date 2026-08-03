# ADR 0010 — Adopt TOON encoding for JSON/text fed into LLM context

**Status:** Accepted (CEO directive 2026-07-31)
**Related:** [`IRON-RULES.md` §38](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md#section-38--toon-encode-jsontext-context-wherever-feasible-ceo-directive-2026-07-31) (`mooniex:IRON-RULES.md`), `reference_headroom_token_tool` memory (superseded — see below)

## Context

CEO shared the TOON project (`toon-format/toon`, MIT — [toonformat.dev](https://toonformat.dev)) after noticing "โปรแกรมทำงานไวขึ้น ลดโทเคนได้จริง" from a demo. TOON is a compact serialization of the JSON data model for LLM prompts: YAML-style indentation for structure + CSV-style rows for uniform arrays. On uniform data (same fields across array items) it collapses repeated keys into one header line + value-only rows — published benchmark: ~42.6% fewer tokens than JSON across 244 retrieval questions on 4 models, with slightly *better* accuracy (72.2% vs 71.4%). Deeply nested / heterogeneous JSON does not benefit and should stay JSON.

Investigation this session found two concrete, high-confidence targets already in the org's own code:

1. **`mooniex-agents` MCP org server** (`runners/cto_mcp_server.py`) — `wiki_search`, `wiki_list`, `get_task`, `list_projects`, `stats` all currently return `json.dumps(..., indent=2)`, some hard-truncated at a fixed char cap (`[:6000]`, `[:8000]`). This is exactly the data that lands in every CTO/DEV Claude Code session's context.
2. **`mooniex-claudeflow`** — every tool-call result in the Kimi K2.6 loop is pushed back via `JSON.stringify(result)` (`src/webhook/claude.js:1511`, mirrored in `agents/base.js:542`, `agents/session.js:117`). This one is **metered** (OpenRouter), not flat-fee, so token count is a direct cost line, not just context pressure. By contrast, `assist/draft.js`'s few-shot examples and `claude.js`'s pending-verify/already-verified context blocks are already hand-formatted as compact text, not raw JSON — no upgrade needed there.

This exact move was proposed and **parked** earlier (see `reference_headroom_token_tool` memory) with an explicit revisit condition: *"revisit at Pro downgrade IF rate-limited."* The org downgraded from Claude Max ($214/mo) to Pro ($20/mo) on 2026-07-23. CEO confirmed on 2026-07-31 that CTO/DEV Claude Code sessions have since hit rate limits "บ่อยมาก" (very often). The condition is met.

## Decision

Adopt TOON as the default serialization for JSON/tabular data fed into any LLM context, wherever the data shape is uniform enough to benefit (arrays/dicts with repeated field shape). Do **not** force TOON onto API-mandated structures (tool-call schemas, the `messages` array itself, non-LLM request bodies) — those stay JSON per protocol contract. Full rule text lives in IRON-RULES §38 (durable, applies org-wide going forward); this ADR is the historical record of why and the initial rollout scope.

### Rollout (initial — tracked via org task queue)
- **mooniex-agents**: add a TOON encode helper (Python — verify official `toon-format/toon-python` port on PyPI before adding as a dependency; do not blindly pick one of the unofficial competing packages) and wire it into `cto_mcp_server.py`'s list/table-shaped tool returns, replacing the `json.dumps(...)[:N]` truncation pattern.
- **mooniex-claudeflow**: add a TOON encode helper (JS — `@toon-format/toon` on npm, confirmed official) and wrap the tool-call result before `JSON.stringify(result)` at the three sites above, falling back to JSON when the result isn't uniform/tabular.

## Consequences

- **Upside:** fewer tokens per CTO/DEV Claude Code turn → more headroom under the tighter Pro-plan rate-limit ceiling; fewer tokens per Kimi tool-call turn in claudeflow → direct OpenRouter cost reduction.
- **Downside / watch-items:** an extra serialization dependency in two repos (supply-chain surface — verify package provenance, per `feedback_api_key_registry`-style registry-first discipline extended to deps); TOON must round-trip losslessly for anything the code re-parses (not just anything a human/LLM reads) — any call site that `JSON.parse`s a tool result downstream needs the decode path wired too, not just encode.
- **Non-goal:** this is not a blanket "replace all JSON everywhere" — nested/heterogeneous structures and API-contract JSON stay as-is per IRON-RULES §38.3.
