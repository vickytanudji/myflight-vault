---
tags: [todo, reminders]
---

# Standing Reminders — Check This Before Signing Off Each Session

## Awaiting a cc-sonnet Run (brief written, not yet executed)
- [ ] **Combined fix brief** (`fix/pilot-transmission-fallback-skip-narrowing-stop-polling-leak`) — fixes 3 things:
  1. `handle_pilot_transmission` returns a generic in-character fallback phrase instead of silence on any LLM failure
  2. `_run_or_skip`'s skip logic narrowed to genuine unreachability only, not a broad exception catch-all
  3. `stop_polling()` now closes a connected-but-never-polled handle on shutdown
  See [[07-Bug-Log]] for full context on each origin.

## Awaiting Live Retest Only (code fixed and merged)
- [ ] **SimConnect connection-robustness fixes** (handle leaks, silent swallow, unguarded read, event-loop stall) — all merged to `develop`, fully pytest-verified. **MSFS 2020 live retest still outstanding:** run the one-liner Python script, then `python -m core.main` with MSFS closed, then launch MSFS, then close/reopen it. See [[SimConnect-Client]].
- [ ] **Connection watchdog + dual-sim docs branch** — code is written and merged, MSFS 2020 live retest still not run.

## Owed Live Tests
- [ ] **Frequency-tuning scenario (e)** — Center-sparse fallback (Enroute) + Departure/Go-Around real-frequency requirement. Needs a longer flight past Ground/Tower Departure. Scenarios (a)-(d) all confirmed. See [[Frequency-Tuning-Retest-Checklist]].
- [ ] **AI/FSLTL traffic probe** (`tools/probe_ai_traffic.py`) — needs the PC; some scenarios need real other traffic (VATSIM session or another human in multiplayer). See [[AI-FSLTL-Traffic]].
- [ ] What camera-state value a cold-and-dark **MSFS 2024** load reports for [[Flight-Load-Gate]] — never tested (2020 confirmed only). The smoke test never actually read `CAMERA STATE`, so this remains open even after the 2024 smoke test passed.

## Content Review Never Actually Done
- [ ] **Enroute/Approach/Go-Around content review** — these three templates' *content* (not phrasing) changed from the old LLM-generated behavior during the templating refactor. Side-by-side review was deferred multiple times, never completed. Fold into the next big test flight.

## Small/Cosmetic, No Urgency
- [ ] Dangling doc references in `frequency_gate.py`/probe scripts pointing at unmerged investigation-branch docs
- [ ] `CLAUDE.md`'s "Active work now (Month 4, Step 1)" paragraph is stale re: current-airport resolution

## Deferred to "Polish Later" Phase (don't re-raise until app is functionally done)
- [ ] ICAO phraseology validation for the 6 PROVISIONAL templates against real Doc 4444/LiveATC
- [ ] Real Facility Data API investigation for anything (Center freq, AI traffic, procedures) — already ruled impractical multiple times; don't re-open without new information

## Vault / Workflow Housekeeping
- [ ] This vault now lives at `github.com/vickytanudji/myflight-vault` (**public**, no auth needed to read) and is edited directly on **Mac** via a filesystem connector, then synced to GitHub, then pulled onto the PC via the Obsidian Git plugin.
- [ ] Any GitHub PAT shared in chat for a push is single-use per session — Claude has no persistent credential storage. Regenerate/revoke tokens after use as a matter of course.
