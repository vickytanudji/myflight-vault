---
tags: [dashboard, current-status]
---

# What's Happening

## Right now
<<<<<<< HEAD
<<<<<<< HEAD
Just closed out a large batch of readback-matching and dispatch-redesign fixes — all confirmed live tonight (2026-09-27). **No new code work currently in flight.** Everything remaining below is either a live test on the PC or explicitly deferred/deprioritized.

## Recently confirmed and merged (2026-09-27)
1. **Readback matching hardened across the board:** frequency format tolerance ("127, decimal 5", "127.5" now match "127.500"), misheard-"decimal" homophone tolerance, and callsign flight-number format tolerance ("842" now matches "eight forty-two"). The callsign bug's real root cause turned out to be the *shared tokenizer* corrupting comma-separated numbers ("842, 10,000" → one broken token), not the callsign matcher itself — found via proper reproduction before assuming the brief's diagnosis was right.
2. **Stale readbacks now clear on an ordinary phase advance**, not just a re-arm (previously DEPARTURE's readback could sit "pending" indefinitely after the flight moved on to APPROACH).
3. **Automatic speed-based takeoff-clearance dispatch replaced with manual-only.** Investigated `ON_ANY_RUNWAY` as a position-based alternative first — confirmed unreliable via two open MSFS bug reports — and correctly stopped rather than building on unconfirmed data. Implemented instead: a WebSocket trigger + a PTT voice phrase ("ready for departure"). Fixes a real bug as a side effect: sessions started airborne no longer fire a takeoff clearance at touchdown.

## Earlier this week, also confirmed and merged
- SimConnect connection-cleanup regression (the `timerThread` bug) — root-caused for real after an initial fix turned out insufficient
- Enroute/Approach/Go-Around content review — live-confirmed across two flights
- APPROACH↔TOWER_ARRIVAL flapping + missed-go-around (on-ground blip) — needed two fix iterations, both now confirmed across two independent flights each
- Two combined bug-fix briefs (pilot-transmission silent-failure fallback, `_run_or_skip` narrowing, stale-handle leaks)

## Next things to do — all on the PC, no new briefs needed
In rough priority order:

1. **Frequency-tuning scenario (e)** — a longer flight past Ground/Tower Departure, to check Center-sparse fallback and Departure/Go-Around's real-frequency requirements.
2. **Full `core.main` connection-robustness retest** — the disconnect side is confirmed; still need to confirm MSFS reconnecting successfully while `core.main` keeps running throughout (not just restarting the app).
3. **MSFS 2024 camera-state check** — what value a cold-and-dark 2024 load reports, for the flight-load gate (2020 confirmed only so far).
4. **AI/FSLTL traffic probe** (`tools/probe_ai_traffic.py`) — first-ever live run, in its own session. Some parts need real other traffic (VATSIM session or another human in multiplayer).

## Deferred, not urgent (explicitly deprioritized 2026-09-27)
- Manual-takeoff-clearance Tests 3 & 4 (airborne-start-bug live retest; go-around/readback-interaction sanity check) — watch for opportunistically, not a priority
- A one-off `DEPARTURE → TOWER_DEPARTURE` transition logged at 4,240ft while climbing — looks like a possible telemetry glitch, only seen once, self-corrected immediately
=======
Just closed out the `phase_detector.py` boundary-flapping and missed-go-around fixes — both confirmed live across two independent real flights each and merged. **No new code work currently in flight.** Everything remaining below is either a live test on the PC or explicitly deferred. A large proactive "phase-boundary noise audit" brief has been written (targeting Claude Code on the web / Opus 5.5 High) but is not tracked here in detail — see your own notes for it.

## Recently confirmed and merged
1. **APPROACH↔TOWER_ARRIVAL flapping + missed go-around (on-ground blip)** — both fixed and live-confirmed twice. The flap fix needed two iterations (a flat hysteresis margin wasn't enough against real-world variance between approaches; the real fix combines a wider magnitude margin with a boundary-specific reversal debounce). The go-around fix correctly recognizes a real go-around even when it includes a brief on-ground touch during a low pass.
2. **SimConnect connection-cleanup regression** — the original leak-fix didn't actually work (`timerThread` AttributeError meant the release action never ran); root-caused and fixed for real, confirmed live with 3 consecutive clean attempts.
3. **Enroute/Approach/Go-Around content review** — the long-owed review finally happened and is now live-confirmed across two flights. Enroute's redundant Center handoff removed, Approach combines descend+clearance+Tower-handoff, Go-Around's vectoring-frequency handoff restored.
4. **Two combined bug-fix briefs** — pilot-transmission silent-failure fallback, `_run_or_skip` test-skip narrowing, Go-Around test was confirmed stale (no real bug ever existed).

## Next things to do — all on the PC, no new briefs needed
In rough priority order:

=======
Just closed out the `phase_detector.py` boundary-flapping and missed-go-around fixes — both confirmed live across two independent real flights each and merged. **No new code work currently in flight.** Everything remaining below is either a live test on the PC or explicitly deferred. A large proactive "phase-boundary noise audit" brief has been written (targeting Claude Code on the web / Opus 5.5 High) but is not tracked here in detail — see your own notes for it.

## Recently confirmed and merged
1. **APPROACH↔TOWER_ARRIVAL flapping + missed go-around (on-ground blip)** — both fixed and live-confirmed twice. The flap fix needed two iterations (a flat hysteresis margin wasn't enough against real-world variance between approaches; the real fix combines a wider magnitude margin with a boundary-specific reversal debounce). The go-around fix correctly recognizes a real go-around even when it includes a brief on-ground touch during a low pass.
2. **SimConnect connection-cleanup regression** — the original leak-fix didn't actually work (`timerThread` AttributeError meant the release action never ran); root-caused and fixed for real, confirmed live with 3 consecutive clean attempts.
3. **Enroute/Approach/Go-Around content review** — the long-owed review finally happened and is now live-confirmed across two flights. Enroute's redundant Center handoff removed, Approach combines descend+clearance+Tower-handoff, Go-Around's vectoring-frequency handoff restored.
4. **Two combined bug-fix briefs** — pilot-transmission silent-failure fallback, `_run_or_skip` test-skip narrowing, Go-Around test was confirmed stale (no real bug ever existed).

## Next things to do — all on the PC, no new briefs needed
In rough priority order:

>>>>>>> origin/main
1. **Full `core.main` connection-robustness retest** — beyond the already-confirmed one-liner: launch → close MSFS → reopen, and specifically a shutdown right after a successful connect but before polling starts.
2. **Frequency-tuning scenario (e)** — a longer flight past Ground/Tower Departure, to check Center-sparse fallback and Departure/Go-Around's real-frequency requirements.
3. **MSFS 2024 camera-state check** — what value a cold-and-dark 2024 load reports, for the flight-load gate (2020 confirmed only so far).
4. **AI/FSLTL traffic probe** (`tools/probe_ai_traffic.py`) — first-ever live run, in its own session. Some parts need real other traffic (VATSIM session or another human in multiplayer).
<<<<<<< HEAD
>>>>>>> origin/main
=======
>>>>>>> origin/main

## Not being worked on right now (deliberately)
- ICAO phraseology validation for the 6 PROVISIONAL templates — deferred to "polish later, after the app is functionally done"
- Real Facility Data API (Center frequencies, AI traffic, procedures) — ruled impractical multiple times, don't re-open without new information
- Go-Around's re-armed second-attempt template — fully built and tested but unreachable from live dispatch (engine's one-shot guard never re-arms) — a separate, known, not-currently-prioritized gap
- `AIRSPEED_ROLLING_HYSTERESIS_KTS` — now likely unneeded given manual-only takeoff dispatch, flagged for a future dedicated cleanup, not removed yet

## Next test session
<<<<<<< HEAD
<<<<<<< HEAD
A full step-by-step guide is ready: [[Next-Test-Session-Guide]] (may need a light refresh given tonight's changes — check against [[09-Standing-Reminders]] for the current authoritative list)
=======
A full step-by-step guide is ready: [[Next-Test-Session-Guide]]
>>>>>>> origin/main
=======
A full step-by-step guide is ready: [[Next-Test-Session-Guide]]
>>>>>>> origin/main

## For full detail
See [[00-Home]] for the full dashboard, [[09-Standing-Reminders]] for the complete checklist with links, [[07-Bug-Log]] for everything fixed this session.
