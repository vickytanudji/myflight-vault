---
tags: [todo, reminders]
---

# Standing Reminders — Check This Before Signing Off Each Session

## Awaiting Live Retest Only (code fixed and merged)
- [ ] **SimConnect connection-robustness fixes** (handle leaks, silent swallow, unguarded read, event-loop stall) — all merged to `develop`, fully pytest-verified. **MSFS 2020 live retest still outstanding:** run the one-liner Python script, then `python -m core.main` with MSFS closed, then launch MSFS, then close/reopen it. See [[SimConnect-Client]].

## New Follow-Up Items (found this session, correctly left out of scope for now)
- [ ] `handle_pilot_transmission` silently swallows every LLM error and returns `""` — a pilot gets total silence on a real LLM failure instead of an "unable/standby" fallback. Worth a future brief.
- [ ] `_run_or_skip`'s skip logic is broader than ideal — skips on any `APIStatusError`/`ValueError`/timeout, not just "no model loaded." A real 500 with a model loaded could silently skip instead of fail.
- [ ] `stop_polling()`'s early-return: a connected-but-never-polled handle isn't closed on shutdown — a different leak scenario than the ones just fixed.

## Owed Live Tests
- [ ] **Frequency-tuning scenario (e)** — Center-sparse fallback (Enroute) + Departure/Go-Around real-frequency requirement. Needs a longer flight past Ground/Tower Departure. Scenarios (a)-(d) all confirmed. See [[Frequency-Tuning-Retest-Checklist]].
- [ ] **AI/FSLTL traffic probe** (`tools/probe_ai_traffic.py`) — needs the PC; some scenarios need real other traffic (VATSIM session or another human in multiplayer). See [[AI-FSLTL-Traffic]].
- [ ] **Connection watchdog + dual-sim docs branch** (`feat/formalize-dual-sim-support`) — code is written and committed, MSFS 2020 live retest (the one-liner python script + the "MSFS closed" check) still not run. See [[SimConnect-Client]].
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
- [ ] This vault now syncs via GitHub (`vickytanudji/myflight-vault`, private) + Obsidian Git plugin. If Claude hasn't pushed in a while, check whether Obsidian Git has pulled the latest before assuming the vault is stale.
- [ ] The PAT used to set this up should be revoked/regenerated after initial setup is confirmed working, since it was shared in plaintext in chat.
