# cookierun-bot — the Cookie Run Classic money farm that learns

**Repo:** `/Users/gob/Projects/cookierun-bot` (Mac) · box checkout `C:\Users\UsEr\cookierun-bot` (winbox, BlueStacks).
**Owner:** CEO plays and answers; CTO runs the loop. Project knowledge lives in the repo:
`knowledge/cookierun-game-mechanics.md` (game facts, tagged [CEO]/[measured]/[fandom]),
`docs/APP-SUPERVISOR.md` (the app is the supervisor and the interface), `docs/DATA-STEWARD.md` (data lifecycle).

## Standing rules (CEO, dated)
- **2026-09-06 — Cookie Run Script is the only interface.** Every workflow the CTO runs is a button in the app
  and a function on its local pipe (`127.0.0.1:8794`); the CTO drives the app through `tools/appctl.py`, the
  window shows `[ท่อ:CTO] เรียก …`. Button first, pipe second, script never alone.
- **2026-09-06 — ESC above everything; no window = no bot.** ESC/F12 kills the bot and holds every launch
  until ▶; the bot runs inside a kill-on-close Job Object and stops itself without the app's heartbeat.
- **2026-09-06 — Data steward.** Warn, never auto-delete (`tools/disksense.py`, daily). Back up to Drive
  (`BACKUP/CookieRun Backup/`, gate row in `playbooks/drive-archive-gate.md`) before deleting; one tar per
  take; takes kept locally as MP4. RunPod volume `cookierun-train` (EU-RO-1, 10 GB) holds only the current
  training tarball. Data stays on the Windows box.
- **Goals (2026-09-06):** survival first (falls > hits), coins are the scoreboard, nights unattended, days tuned
  together; a richer stage only when EP1 hits ≈ 0.

## State on 2026-09-06
- Champion **v5**; **v7** (takes ×3 + dagger ×4, 112,679 rows, $0.54) beat v5 on the combined 14 A/B rounds
  (+6.6 % coins, 3.5 vs 3.9 hits) but not on either 8-round session alone — promotion is the CEO's call.
- Falls are now first-class: the «Save the Cookie … Pit Lift» band is the fall signal (3/3 measured);
  `rounds.jsonl` carries `falls` and `deaths`; a candidate that falls more is never promoted.
- Open: exe rebuild with the repo-path fix; drive.file re-auth; ☁ RunPod training as a button; worker under the app.
