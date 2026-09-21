---
tags: [architecture, phase-detector]
---

# Phase Detector (`core/atc/phase_detector.py`)

Thread-safe state machine. `PhaseDetector.update(flight_data)` is called once per poll (0.5s interval) and returns the current confirmed `FlightPhase`.

> [!warning] High-risk file
> Per project's own hard rule: **never merge anything touching this file without a live MSFS retest.** This file has had the exact same critical bug (altitude-continuity freeze) silently reintroduced three separate times historically.

## CLEARANCE / GROUND / PARKED Boundary (redesigned this session)

**Old (buggy) model:** raw ground-speed threshold. A stationary aircraft with engines running produced enough sensor noise to flap across the 1.0kt threshold repeatedly, disrupting readback-gating.

**New (current) model**, based on real power/engine state:
- **PARKED:** no engines, no ground/APU power
- **CLEARANCE:** on ground, engines **not** running, external power **or** APU active
- **GROUND:** on ground, engines **running** (regardless of speed)

### Power-state debounce
`POWER_STATE_DEBOUNCE_POLLS = 5` — a new power-state reading must hold for 5 consecutive accepted polls before a PARKED↔CLEARANCE transition confirms. Fixed a secondary flap discovered after the main speed-based fix (see [[07-Bug-Log]]).

### Power-telemetry fallback
If `external_power_on`/`apu_pct_rpm` never populate at all after 60 consecutive on-ground, engines-off polls (~20s), the detector **falls back to treating this as CLEARANCE** rather than staying stuck in PARKED forever. Logged clearly. This is expected on some aircraft/liveries where this telemetry simply isn't exposed.

> [!info] Known limitation, accepted
> External-power/APU telemetry reliability varies per aircraft — same class of issue as `ATC_AIRLINE`/`ATC_FLIGHT_NUMBER` not being set on some liveries. Not fixable in general; the fallback exists specifically for this.

## Existing Mechanisms (pre-dating this session, still load-bearing)
- **Altitude-continuity rolling baseline** — the actual fix for the historic 3x-recurring altitude-freeze bug. Baseline rolls forward on every accepted poll.
- **`ALTITUDE_CONTINUITY_RECOVERY_STREAK`** — after 10 consecutive rejections, force-accept the next poll as a new baseline (safety net, not the primary fix).
- **Go-around detection** — `GO_AROUND_CONFIRM_POLLS = 5`, Rule 0 holds phase based on the *current poll's own* vs/on-ground signature (not accumulated streak) — this distinction is the actual fix for the historic go-around priority race bug.
- **Exact-zero quorum gate** — rejects polls where ≥3 of {gs, ias, alt, vs} are exactly 0.0 simultaneously while engines running (catches implausible sensor glitches).

## Rule Table (unchanged this session except Rules 1–3)
| Rule | Condition | Result |
|---|---|---|
| 0 | current phase ∈ GO_AROUND_ELIGIBLE, airborne, vs > climb threshold | hold current phase |
| 1 | on ground, no engines, no ground/APU power | PARKED |
| 2 | on ground, no engines, ground/APU power active (debounced) | CLEARANCE |
| 3 | on ground, engines running | GROUND |
| 4 | on ground, ias ≥ 40kt | TOWER_DEPARTURE |
| 5–8 | airborne rules (unchanged) | DEPARTURE / ENROUTE / APPROACH / TOWER_ARRIVAL |

## See Also
- [[Flight-Load-Gate]] — gates *when* the detector starts being fed real data at all
- [[07-Bug-Log]] — full history of bugs found in this file
