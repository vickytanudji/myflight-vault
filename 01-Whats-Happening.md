---
tags: [dashboard, current-status]
---

# What's Happening

## Right now
Just closed out the `phase_detector.py` boundary-flapping and missed-go-around fixes — both confirmed live across two independent real flights each and merged. **No new code work currently in flight.** Everything remaining below is either a live test on the PC or explicitly deferred. A large proactive "phase-boundary noise audit" brief has been written (targeting Claude Code on the web / Opus 5.5 High) but is not tracked here in detail — see your own notes for it.

## Recently confirmed and merged
1. **APPROACH↔TOWER_ARRIVAL flapping + missed go-around (on-ground blip)** — both fixed and live-confirmed twice. The flap fix needed two iterations (a flat hysteresis margin wasn't enough against real-world variance between approaches; the real fix combines a wider magnitude margin with a boundary-specific reversal debounce). The go-around fix correctly recognizes a real go-around even when it includes a brief on-ground touch during a low pass.
2. **SimConnect connection-cleanup regression** — the original leak-fix didn't actually work (`timerThread` AttributeError meant the release action never ran); root-caused and fixed for real, confirmed live with 3 consecutive clean attempts.
3. **Enroute/Approach/Go-Around content review** — the long-owed review finally happened and is now live-confirmed across two flights. Enroute's redundant Center handoff removed, Approach combines descend+clearance+Tower-handoff, Go-Around's vectoring-frequency handoff restored.
4. **Two combined bug-fix briefs** — pilot-transmission silent-failure fallback, `_run_or_skip` test-skip narrowing, Go-Around test was confirmed stale (no real bug ever existed).

## Next things to do — all on the PC, no new briefs needed
In rough priority order:

1. **Full `core.main` connection-robustness retest** — beyond the already-confirmed one-liner: launch → close MSFS → reopen, and specifically a shutdown right after a successful connect but before polling starts.
2. **Frequency-tuning scenario (e)** — a longer flight past Ground/Tower Departure, to check Center-sparse fallback and Departure/Go-Around's real-frequency requirements.
3. **MSFS 2024 camera-state check** — what value a cold-and-dark 2024 load reports, for the flight-load gate (2020 confirmed only so far).
4. **AI/FSLTL traffic probe** (`tools/probe_ai_traffic.py`) — first-ever live run, in its own session. Some parts need real other traffic (VATSIM session or another human in multiplayer).

## Not being worked on right now (deliberately)
- ICAO phraseology validation for the 6 PROVISIONAL templates — deferred to "polish later, after the app is functionally done"
- Real Facility Data API (Center frequencies, AI traffic, procedures) — ruled impractical multiple times, don't re-open without new information
- Go-Around's re-armed second-attempt template — fully built and tested but unreachable from live dispatch (engine's one-shot guard never re-arms) — a separate, known, not-currently-prioritized gap

## Next test session
A full step-by-step guide is ready: [[Next-Test-Session-Guide]]

## For full detail
See [[00-Home]] for the full dashboard, [[09-Standing-Reminders]] for the complete checklist with links, [[07-Bug-Log]] for everything fixed this session.
