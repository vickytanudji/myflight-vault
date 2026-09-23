---
tags: [dashboard]
---

# MyFlight — Project Home

AI-generated, phraseology-accurate ATC add-on for Microsoft Flight Simulator. Polls live SimConnect flight data, detects flight phase, generates real ICAO-phraseology ATC transmissions via a local LLM, spoken via TTS, with PTT-triggered pilot voice input via STT.

## Quick Links
- [[01-Whats-Happening|📍 What's Happening — start here]]
- [[01-Overview|Project Overview]]
- [[02-Roadmap|Roadmap & Progress]]
- [[09-Standing-Reminders|⚠️ Standing Reminders — check before you sign off]]
- [[08-Gotchas-And-Quirks|Gotchas & Quirks]]
- [[07-Bug-Log|Bug Log]]
- [[10-Workflow|Git / cc-sonnet Workflow]]

## Architecture
- [[Phase-Detector]]
- [[ATC-Engine]]
- [[Readback-Gating]]
- [[Frequency-Tuned-Dispatch]]
- [[Flight-Load-Gate]]
- [[SimConnect-Client]]

## Investigations (all closed/decided)
- [[Center-Frequencies|Real Center/ARTCC Frequencies — NO-GO]]
- [[ChartFox-Procedures-Taxiways|ChartFox Procedures + Taxiways — NO-GO / DEFER]]
- [[AI-FSLTL-Traffic|AI/FSLTL Traffic Awareness — PURSUE (pending live test)]]
- [[MSFS-2024-Support|MSFS 2024 Support — PURSUE (confirmed working)]]

## Testing
- [[Next-Test-Session-Guide|✅ Next Test Session — step by step]]
- [[Test-Setup-And-Methodology]]
- [[Frequency-Tuning-Retest-Checklist]]

## Current Status Snapshot
Test environment: YSSY, manual callsign override cleared (real callsign path), PTT key `grave`, cold-and-dark start required.

**Confirmed working live (MSFS 2020):** flight-load gate (incl. manual-trigger interaction), frequency-tuned dispatch (scenarios a–d), readback-gating levels 1–3, manual clearance trigger, `ATC_RUNWAY_AIRPORT_NAME` display-name fix.

**Confirmed via smoke test (MSFS 2024):** all 25+ SimVars identical to 2020, dual-support formalized in docs.

**✅ Connection-cleanup regression FIXED AND LIVE-CONFIRMED (2026-09-23).** A live retest of the earlier connection-robustness work found the original leak-fix didn't actually work — the wheel's `exit()` threw `AttributeError` on `self.timerThread.join()` before ever reaching `self.dll.Close(...)`, so the original fix's `except AttributeError` caught the symptom but never actually released the handle. Fixed: `_release_simconnect()` now calls `sm.dll.Close(sm.hSimConnect)` directly when `timerThread` is absent. **Live retest: 3 consecutive MSFS-closed connection attempts, zero `timerThread` errors, fast clean failures (~0.44-0.47s) each time.** This is the real fix — confirmed live, not just pytest-verified. Merged to `develop`. See [[SimConnect-Client]] for full detail.

**Enroute/Approach/Go-Around content review is DONE — the long-deferred review finally happened, and real fixes landed.** Enroute's redundant Center handoff removed (Departure already does it, real content restored: cruise-altitude check-in). Approach now combines descend instruction + clearance + Tower handoff in one transmission (PROVISIONAL, documented as a scoping decision matching this project's one-transmission-per-phase pattern, not a Doc 4444 citation). Go-Around's vectoring-frequency handoff restored on the first attempt (was a real functional gap — a pilot going around needs to know what frequency to contact). 1219 tests passing. **Live retest owed** — see [[09-Standing-Reminders]].

**All bugs from both combined fix briefs are resolved and merged**: Go-Around test was stale (no real bug), LLM-skip gap fixed, pilot-transmission silent-failure fallback fixed, `_run_or_skip` narrowed correctly.

**Still owed:** frequency-tuning scenario (e) on a longer flight; AI/FSLTL traffic probe live run; MSFS 2024 camera-state value for the flight-load gate; the Enroute/Approach/Go-Around content-fix live retest.

**Vault sync:** this vault lives at `github.com/vickytanudji/myflight-vault` (public), synced between Mac and PC via Obsidian Git. **⚠️ Known risk:** editing the vault from both machines without pushing/pulling in between can produce real git merge conflicts (this happened once — resolved manually). Push after any edit session before switching machines.
