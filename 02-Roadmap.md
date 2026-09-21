---
tags: [roadmap, progress]
---

# Roadmap & Progress

## Month 4 — ATC Phase 1 (Clearance, Ground, Tower Departure)

| Item | Status |
|---|---|
| Full voice exchange, cold-and-dark to airborne | ✅ Confirmed repeatedly |
| Real SID name via ChartFox in Clearance | ✅ Confirmed (wind-based runway selection + real ChartFox SID) |
| Same runway across Clearance/Ground/Tower Departure | ✅ Confirmed |
| ICAO phraseology verified against real recordings | 🔲 Deferred — "polish later" |
| LLM doesn't hallucinate/break character | ✅ **Adversarially tested** — 2 real bugs found & fixed (value fabrication, out-of-authority compliance in `handle_pilot_transmission`) |
| PTT-release-to-voice latency (3–5s target) | ❌ Failing (2–20+s measured) — **accepted as a model-performance issue, deferred to cloud/faster-model fix later** |

## Month 5 — ATC Phase 2 (Enroute, Approach, Tower Arrival, Go-Around)

| Item | Status |
|---|---|
| Full gate-to-gate coherent ATC | ✅ Confirmed |
| Real STAR/approach names via ChartFox | ✅ Confirmed |
| Go-around handling | ✅ Confirmed end-to-end |
| No duplicate/contradictory instructions | ✅ Confirmed |
| Realistic frequency handoff timing | ✅ Confirmed |

## The 5 "Big Items" (in priority order)

### 1. ✅ Realistic Frequency Tuning — IMPLEMENTED, mostly retested
VATSIM-style: pilot must tune the correct real frequency to receive a transmission. See [[Frequency-Tuned-Dispatch]].
- Scenarios (a)–(d) confirmed live on MSFS 2020.
- **Scenario (e) (Center-sparse fallback, Departure/Go-Around real-frequency requirement) still owed — do on a longer flight.**

### 2. ✅ Real Center/ARTCC Frequencies — CLOSED, NO-GO
See [[Center-Frequencies]]. No viable real data source exists. Center frequency stays synthetic permanently. This is an **accepted, permanent limitation**, not a bug.

### 3. ✅ ChartFox Real SID/STAR Procedures + Taxiway Routing — CLOSED
See [[ChartFox-Procedures-Taxiways]].
- **Procedure content: NO-GO** (ChartFox doesn't have it; app has no flight-plan reading anyway, so it wouldn't pay off)
- **Taxiway routing: DEFER** (real data exists via OSM / X-Plane Gateway, but naming coverage is only 43–76%, licensing needs review, and it may not match the sim's actual layout)

### 4. 🟡 AI/FSLTL Traffic Awareness — INVESTIGATED, PURSUE, live test pending
See [[AI-FSLTL-Traffic]]. Native AI traffic and FSLTL collapse into **one problem** — both readable via `SimConnect_RequestDataOnSimObjectType`. Probe script written (`tools/probe_ai_traffic.py`), **never run live**. This is a standing reminder — see [[09-Standing-Reminders]].

### 5. 🟡 MSFS 2024 Support — INVESTIGATED + SMOKE TESTED, confirmed working
See [[MSFS-2024-Support]]. All SimVars identical between 2020/2024. Live smoke test on real MSFS 2024: 30/30 then 28/30 PASS (2 expected/benign). **Formalized as dual-supported** in the connection-watchdog + docs brief.

## Smaller Items Still Open
- [ ] Enroute/Approach/Go-Around **content** review (not just phrasing) — flagged repeatedly, never actually done. Fold into next big test flight.
- [ ] Frequency-tuning scenario (e) — Center-sparse fallback + Departure/Go-Around real frequencies, needs a longer flight
- [ ] AI/FSLTL traffic probe live run (needs PC, possibly real other traffic/VATSIM session)
- [ ] Doc 4444/LiveATC validation of 6 PROVISIONAL phraseology templates — deferred to polish phase
- [ ] Dangling doc references in `frequency_gate.py` / probe scripts pointing at unmerged investigation-branch docs (cosmetic)
- [ ] `CLAUDE.md`'s "Active work now (Month 4, Step 1)" paragraph is stale re: current-airport resolution (now solved via GPS-based `resolve_departure_icao`)
- [ ] **New (flagged, not yet briefed):** prevent the entire process (incl. ATIS) from starting until a real flight loads — ~~this is now DONE~~ see [[Flight-Load-Gate]] ✅ confirmed live
