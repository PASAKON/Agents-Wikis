# ADR: Spawn-chain hardening (CEO→CTO→CXO→DEV)

- **Date:** 2026-06-10
- **Author:** CTO
- **Status:** shipped — mooniex-agents `52556bc` (on top of `f1c0130` live tab titles)

## Context

CEO requested a full audit of the agent spawn pipeline: `spawn-cto.sh` →
`cto-claude.sh`, `send_to_cxo --spawn` → `cxo-claude.sh`, and
`delegate_task` → iTerm/tmux DEV tabs. 13 defects found, all fixed in one
hardening commit; tests 22/22 + 6/6 PASS.

## Key decisions

1. **Unclaimed-spawn watchdog** (`tools/delegate.py:_verify_claimed`):
   25s after an iTerm DEV spawn, a task still `pending`+unclaimed ⇒ close
   stale tab, ONE auto respawn + kickoff, then loud error. Replaces the
   manual SOP "reset to pending + close stale tab before re-firing".
2. **Cross-CTO ownership gate** (`cto_mcp_server._is_mine/_foreign_msg`):
   delegate/merge/reopen/revert refuse tasks owned by another CTO session.
   Unowned tasks + id-less sessions stay unrestricted (legacy/CXO).
3. **Per-CTO log routing**: per-session prompt-hook cursor
   (`state/.cto_log_pos-<sid>`, init at EOF) + `owner_cto` line filter;
   notify + dev-reply hook dual-write `state/logs/cto-<sid>.log`;
   CTO-event lines tagged `[sid]`. Global `cto.log` kept as compat bus.
4. **Idle-ping watcher = ephemeral-only** (`cxo-claude.sh` gates on
   `--session`): primary CEO↔CXO tabs are never pinged/auto-closed.
5. **Thai-safe topic slugs** (`send_to_cxo`): pure-Thai messages hash to
   `t-<sha1:8>` so distinct topics never dedupe-collide into one tab.
6. **Status-write discipline**: `db.set_fields()` for column updates that
   must not touch `status` (tmux/ttyd metadata) — prevents claim regression.

## Deferred (next wave)

- Collapse `cto-claude.sh` into `cxo-claude.sh --role cto` (drift risk
  remains; ALLOWED lists synced + cross-referenced for now).
- `owner_cxo` stamping for CMO/CGO/CFO-delegated tasks (today owner is
  CTO-only; CXO-spawned DEV tabs cluster under a CTO window and replies
  hit only the global log).
- idle-ping watcher: poll the session by tab match instead of
  `current tab of window`; tighten the English-"done" close heuristic.
- Orphaned DEV replies (`state/orphan-dev-replies-*.log`) have no
  re-delivery path — surface into next CTO session via hook.
