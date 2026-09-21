---
tags: [todo, reminders]
---

# Standing Reminders — Check This Before Signing Off Each Session

## Owed Live Tests
- [ ] **Frequency-tuning scenario (e)** — Center-sparse fallback (Enroute) + Departure/Go-Around real-frequency requirement. Needs a longer flight past Ground/Tower Departure. See [[Frequency-Tuning-Retest-Checklist]].
- [ ] **AI/FSLTL traffic probe** (`tools/probe_ai_traffic.py`) — needs the PC; some scenarios need real other traffic (VATSIM session or another human in multiplayer). See [[AI-FSLTL-Traffic]].

## Content Review Never Actually Done
- [ ] **Enroute/Approach/Go-Around content review** — these three templates' *content* (not phrasing) changed from the old LLM-generated behavior during the templating refactor. Side-by-side review was deferred multiple times, never completed. Fold into the next big test flight.

## Next Brief to Write
- [ ] **Connection watchdog timeout in production `client.py`** — the MSFS 2024 smoke test script added a 10s timeout for its own diagnostic purposes; production code still has the underlying unbounded wait. This was the brief in progress at end of last session — see [[SimConnect-Client]] for full context, and the git workflow doc for the exact brief text if it wasn't finished.

## Small/Cosmetic, No Urgency
- [ ] Dangling doc references in `frequency_gate.py`/probe scripts pointing at unmerged investigation-branch docs
- [ ] `CLAUDE.md`'s "Active work now (Month 4, Step 1)" paragraph is stale re: current-airport resolution
- [ ] `stop_polling()`'s silent `except Exception: pass`, and the second handle-leak branch (see [[SimConnect-Client]])

## Deferred to "Polish Later" Phase (don't re-raise until app is functionally done)
- [ ] ICAO phraseology validation for the 6 PROVISIONAL templates against real Doc 4444/LiveATC
- [ ] Real Facility Data API investigation for anything (Center freq, AI traffic, procedures) — already ruled impractical multiple times; don't re-open without new information

## Confirm When MSFS 2024 Work Resumes
- [ ] What camera-state value a cold-and-dark MSFS 2024 load reports (never tested — 2020 only so far)
