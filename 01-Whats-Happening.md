---
tags: [dashboard, current-status]
---

# What's Happening

## Right now
Just closed out a large batch of readback-matching and dispatch-redesign fixes — all confirmed live tonight (2026-09-27). **No new code work currently in flight.** Everything remaining below is either a live test on the PC or explicitly deferred/deprioritized.

## Recently confirmed and merged (2026-09-27)
1. **Readback matching hardened across the board:** frequency format tolerance ("127, decimal 5", "127.5" now match "127.500"), misheard-"decimal" homophone tolerance, and callsign flight-number format tolerance ("842" now matches "eight forty-two"). The callsign bug's real root cause turned out to be the *shared tokenizer* corrupting comma-separated numbers ("842, 10,000" → one broken token), not the callsign matcher itself — found via proper reproduction before assuming the brief's diagnosis was right.
2. **Stale readbacks now clear on an ordinary phase advance**, not just a re-arm (previously DEPARTURE's readback could sit "pending" indefinitely after the flight moved on to APPROACH).
3. **Automatic speed-based takeoff-clearance dispatch replaced with manual-only.** Investigated `ON_ANY_RUNWAY` as a position-based alternative first — confirmed unreliable via two open MSFS bug reports — and correctly stopped rather than building on unconfirmed data. Implemented instead: a WebSocket trigger + a PTT voice phrase ("ready for departure"). Fixes a real bug as a side effect: sessions started airborne no longer fire a takeoff clearance at touchdown.
4. **MSFS 2024 cold-and-dark camera-state confirmed** — reads `CAMERA STATE = 2`, same as 2020's cockpit value. The flight-load gate's `{2, 3}` design works correctly on both sims, no code change needed.

## Earlier this week, also confirmed and merged
- SimConnect connection-cleanup regression (the `timerThread` bug) — root-caused for real after an initial fix turned out insufficient
- Enroute/Approach/Go-Around content review — live-confirmed across two flights
- APPROACH↔TOWER_ARRIVAL flapping + missed-go-around (on-ground blip) — needed two fix iterations, both now confirmed across two independent flights each
- Two combined bug-fix briefs (pilot-transmission silent-failure fallback, `_run_or_skip` narrowing, stale-handle leaks)

## Next things to do — all on the PC, no new briefs needed
In rough priority order:

1. **Frequency-tuning scenario (e)** — a longer flight past Ground/Tower Departure, to check Center-sparse fallback and Departure/Go-Around's real-frequency requirements.
2. **Full `core.main` connection-robustness retest** — the disconnect side is confirmed; still need to confirm MSFS reconnecting successfully while `core.main` keeps running throughout (not just restarting the app).
3. **AI/FSLTL traffic probe** — a minimal, default-OFF, passive/logging-only V1 has been implemented (not yet reviewed/merged); needs a live PC session with real AI traffic density to get the first-ever real data point.

## Deferred, not urgent (explicitly deprioritized 2026-09-27)
- Manual-takeoff-clearance Tests 3 & 4 (airborne-start-bug live retest; go-around/readback-interaction sanity check) — watch for opportunistically, not a priority
- A one-off `DEPARTURE → TOWER_DEPARTURE` transition logged at 4,240ft while climbing — looks like a possible telemetry glitch, only seen once, self-corrected immediately

## Not being worked on right now (deliberately)
- ICAO phraseology validation for the 6 PROVISIONAL templates — deferred to "polish later, after the app is functionally done"
- Real Facility Data API (Center frequencies, AI traffic, procedures) — ruled impractical multiple times, don't re-open without new information
- Go-Around's re-armed second-attempt template — fully built and tested but unreachable from live dispatch (engine's one-shot guard never re-arms) — a separate, known, not-currently-prioritized gap
- `AIRSPEED_ROLLING_HYSTERESIS_KTS` — now likely unneeded given manual-only takeoff dispatch, flagged for a future dedicated cleanup, not removed yet

## Next test session
A full step-by-step guide is ready: [[Next-Test-Session-Guide]] (may need a light refresh given tonight's changes — check against [[09-Standing-Reminders]] for the current authoritative list)

## For full detail
See [[00-Home]] for the full dashboard, [[09-Standing-Reminders]] for the complete checklist with links, [[07-Bug-Log]] for everything fixed this session.
