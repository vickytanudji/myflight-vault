---
tags: [dashboard, current-status]
---

# What's Happening

## Right now
Just finished a long content-and-robustness pass. **No new code work in flight** — everything remaining below is either a live test on the PC or explicitly deferred. This is a natural pause point.

## What just got done (all merged, not yet live-tested)
1. **Enroute/Approach/Go-Around content review** — the long-owed review finally happened. Real fixes landed: Enroute's redundant Center handoff removed, Approach combines descend+clearance+Tower-handoff, Go-Around's vectoring-frequency handoff restored.
2. **SimConnect connection robustness** — 6 sub-fixes (handle leaks, silent swallow, unguarded read, event-loop stall, watchdog timeout, never-polled-handle) across two briefs. 102 tests passing.
3. **Two combined bug-fix briefs** — pilot-transmission silent-failure fallback, `_run_or_skip` test-skip narrowing, Go-Around test was confirmed stale (no real bug ever existed).

## Next 5 things to do — all on the PC, no new briefs needed
In rough priority order:

1. **SimConnect connection-robustness retest** — the one-liner script, then `python -m core.main` with MSFS closed → launched → closed/reopened, specifically testing a shutdown right after connect (before polling starts).
2. **Enroute/Approach/Go-Around content retest** — fly to Enroute and confirm the cruise check-in + single Departure→Center handoff; confirm Approach's combined instruction; confirm Go-Around's frequency is audible on the first attempt.
3. **Frequency-tuning scenario (e)** — a longer flight past Ground/Tower Departure, to check Center-sparse fallback and Departure/Go-Around's real-frequency requirements.
4. **AI/FSLTL traffic probe** (`tools/probe_ai_traffic.py`) — first-ever live run. Some parts need real other traffic (VATSIM session or another human in multiplayer).
5. **MSFS 2024 camera-state check** — what value a cold-and-dark 2024 load reports, for the flight-load gate (2020 confirmed only so far).

## Not being worked on right now (deliberately)
- ICAO phraseology validation for the 6 PROVISIONAL templates — deferred to "polish later, after the app is functionally done"
- Real Facility Data API (Center frequencies, AI traffic, procedures) — ruled impractical multiple times, don't re-open without new information
- Go-Around's re-armed second-attempt template — fully built and tested but unreachable from live dispatch (engine's one-shot guard never re-arms) — a separate, known, not-currently-prioritized gap

## For full detail
See [[00-Home]] for the full dashboard, [[09-Standing-Reminders]] for the complete checklist with links, [[07-Bug-Log]] for everything fixed this session.
