---
tags: [todo, reminders]
---

# Standing Reminders — Check This Before Signing Off Each Session

## Awaiting a cc-sonnet Run (brief already written, not yet executed)
- [ ] **Combined fix brief** (`fix/go-around-freq-llm-skip-connection-robustness`) — fixes 3 things found tonight:
  1. `test_go_around_free_response_does_not_invent_squawk` — `TypeError: _resolve_vectors_freq() missing 2 required positional arguments` — real pre-existing bug or stale test, root cause not yet determined
  2. `requires_real_llm` skip decorator doesn't handle "LM Studio reachable but no model loaded" (400 error) — was causing false test failures instead of clean skips
  3. Three SimConnect connection-robustness gaps: adjacent handle leak (`AircraftRequests()`/`Request()` raising after a successful `SimConnect()`), `stop_polling()`'s silent `except Exception: pass`, unguarded verification read, and moving `reconnect_loop`'s `connect()` call to `asyncio.to_thread` to stop event-loop stalls
  See [[07-Bug-Log]] for full detail on each.

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
