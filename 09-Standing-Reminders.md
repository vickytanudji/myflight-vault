---
tags: [todo, reminders]
---

# Standing Reminders — Check This Before Signing Off Each Session

## Awaiting Live Retest Only (all code fixed and merged)
- [ ] **SimConnect connection-robustness fixes** (handle leaks incl. the never-polled-handle case, silent swallow, unguarded read, event-loop stall, watchdog timeout) — all merged to `develop`, fully pytest-verified (1218 passed, 16 pre-existing skips). **MSFS 2020 live retest still outstanding for all of it:** run the one-liner Python script, then `python -m core.main` with MSFS closed, then launch MSFS, then close/reopen it — specifically also test a shutdown right after a successful connect (before `start_polling()`), to exercise the newest fix. See [[SimConnect-Client]].

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
- [ ] This vault lives at `github.com/vickytanudji/myflight-vault` (**public**, no auth needed to read) and is now edited **directly on Mac** via a filesystem connector — no more GitHub round-trip needed for Claude to update it. Push to GitHub / pull onto the PC via Obsidian Git remains the user's responsibility.
- [ ] Any GitHub PAT shared in chat for a push is single-use per session — Claude has no persistent credential storage. Regenerate/revoke tokens after use as a matter of course.
