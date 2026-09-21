---
tags: [dashboard]
---

# MyFlight — Project Home

AI-generated, phraseology-accurate ATC add-on for Microsoft Flight Simulator. Polls live SimConnect flight data, detects flight phase, generates real ICAO-phraseology ATC transmissions via a local LLM, spoken via TTS, with PTT-triggered pilot voice input via STT.

## Quick Links
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
- [[Test-Setup-And-Methodology]]
- [[Frequency-Tuning-Retest-Checklist]]

## Current Status Snapshot
Test environment: YSSY, manual callsign override cleared (real callsign path), PTT key `grave`, cold-and-dark start required.

**Confirmed working live (MSFS 2020):** flight-load gate (incl. manual-trigger interaction), frequency-tuned dispatch (scenarios a–d), readback-gating levels 1–3, manual clearance trigger, `ATC_RUNWAY_AIRPORT_NAME` display-name fix.

**Confirmed via smoke test (MSFS 2024):** all 25+ SimVars identical to 2020, dual-support formalized in docs.

**Code written, not yet live-tested:** connection watchdog timeout (2020 retest owed).

**Code written, not yet even run by cc-sonnet:** combined fix for the Go-Around signature bug, the LLM-skip-decorator gap, and 3 connection-robustness gaps — see [[09-Standing-Reminders]].

**Still owed:** frequency-tuning scenario (e) on a longer flight; AI/FSLTL traffic probe live run; Enroute/Approach/Go-Around content review; MSFS 2024 camera-state value for the flight-load gate.

**Vault sync:** this vault now lives at `github.com/vickytanudji/myflight-vault` (private), synced to the PC via the Obsidian Git plugin.
