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

**⚠️ REGRESSION FOUND (live retest, 2026-09-23) — root cause identified, fix written, LIVE RETEST STILL PENDING.** Root cause: the wheel's `exit()` throws `AttributeError` on `self.timerThread.join()` BEFORE reaching `self.dll.Close(...)` on the next line — so the original fix's `except AttributeError` caught the symptom but the actual release action never ran; the handle was still genuinely leaked, just silently. Fix: `_release_simconnect()` now calls `sm.dll.Close(sm.hSimConnect)` directly when `timerThread` is absent, bypassing `exit()` entirely (`dll`/`hSimConnect` are always set in `__init__`, safe to call). Test gap found too: the old fake object had no `dll`/`hSimConnect` at all, so no test could ever have asserted the real release happened — rebuilt to mirror the real wheel's state, now asserts `dll.Close.assert_called_once_with(...)`. **This must NOT be trusted until the live retest (3+ consecutive MSFS-closed attempts, zero timerThread errors) actually passes — this exact bug already went through one full "fixed → merged → still broken live" cycle.** See [[SimConnect-Client]] and [[09-Standing-Reminders]].

**Enroute/Approach/Go-Around content review is DONE — the long-deferred review finally happened, and real fixes landed.** Enroute's redundant Center handoff removed (Departure already does it, real content restored: cruise-altitude check-in). Approach now combines descend instruction + clearance + Tower handoff in one transmission (PROVISIONAL, documented as a scoping decision matching this project's one-transmission-per-phase pattern, not a Doc 4444 citation). Go-Around's vectoring-frequency handoff restored on the first attempt (was a real functional gap — a pilot going around needs to know what frequency to contact). 1219 tests passing. **Live retest owed** — see [[09-Standing-Reminders]].

**All bugs from both combined fix briefs are resolved and merged**: Go-Around test was stale (no real bug), LLM-skip gap fixed, pilot-transmission silent-failure fallback fixed, `_run_or_skip` narrowed correctly.

**Still owed:** frequency-tuning scenario (e) on a longer flight; AI/FSLTL traffic probe live run; MSFS 2024 camera-state value for the flight-load gate; the connection-robustness live retest; the Enroute/Approach/Go-Around content-fix live retest.

**Vault sync:** this vault lives at `github.com/vickytanudji/myflight-vault` (public). **Claude now edits it directly on Mac** via a filesystem connector — no GitHub round-trip needed for updates. Pushing to GitHub / pulling onto the PC via Obsidian Git remains a manual step for the user.
