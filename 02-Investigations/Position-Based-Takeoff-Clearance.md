---
tags: [investigation, closed]
status: CLOSED — ON_ANY_RUNWAY rejected, manual-only design implemented
---

# Investigation: Position-Based Takeoff Clearance

## Question
Should GROUND → TOWER_DEPARTURE dispatch of the takeoff-clearance transmission move from a speed threshold (40kt IAS) to something position-based (on the runway) and/or manual (an explicit pilot request), matching how real ATC actually works?

## Trigger
The B2 phase-boundary-audit fix (see [[Phase-Detector]]) hardened the speed threshold against *repeated flapping*, but explicitly left a known, accepted residual gap: a single spurious `GROUND → TOWER_DEPARTURE` on a windy, fast taxi-out could still fire a premature takeoff clearance mid-taxi. The user explicitly requested a redesign eliminating speed as a trigger entirely.

## Findings

### `ON_ANY_RUNWAY` — confirmed unreliable, investigation stopped here
- **Missing from the pinned `SimConnect==0.4.26` wrapper table** (confirmed by reading `RequestList.py` in the wheel itself) — would need the same raw `SimConnect.Request` bypass already established for other missing fields (`ATC_RUNWAY_AIRPORT_NAME`, `EXTERNAL_POWER_ON`, `CAMERA_STATE`).
- **Two open MSFS DevSupport bug reports found**, matching this exact use case:
  - [Topic 3975](https://devsupport.flightsimulator.com/t/a-var-on-any-runway-doesnt-seem-to-be-working-correctly/3975): on MSFS 2020, reads `False` after a gate start, even while genuinely on a runway.
  - [Topic 16895](https://devsupport.flightsimulator.com/t/su4beta-1-6-18-0-on-any-runway-returns-false/16895): on the MSFS 2024 SU4 beta, returns `False` for any aircraft.
- Whether either bug is fixed in current builds is unknown; direct SDK documentation and the forum threads themselves were unreachable from the investigation environment.
- **Investigation correctly stopped at this gate** rather than implementing against an unconfirmed data source — matching this project's established discipline for every previous "is this SimVar actually reliable" question.

### Recommended design: manual-only, no new SimVar needed
The investigation's own reasoning: **a real pilot requests takeoff from the holding point, off the runway** — so an "on runway" check would actually *reject* the correct, real-world request timing. Position data isn't just unreliable here; it's the wrong signal even if it worked.

**Recommendation:** remove automatic dispatch entirely. Replace with:
1. A new `request_takeoff_clearance` WebSocket trigger (mirrors the existing `request_clearance` pattern)
2. A PTT-spoken phrase ("ready for departure" and reasonable variants) — necessary because no Electron/UI app exists yet to send the WebSocket message directly from a normal user workflow

**Why manual-only, not automatic+manual:** the only automatic signals available are speed (which the user wants eliminated) or position (which fires in the wrong order relative to real ATC, even before considering reliability). Neither is a good automatic trigger, so automatic dispatch was dropped rather than kept alongside manual.

### Phase-label vs. dispatch-timing separation
Confirmed: the phase label (GROUND vs TOWER_DEPARTURE) is still needed elsewhere — go-around graze detection, stale-transmission ordering — and was left completely untouched. Only the *transmission-firing* side effect was removed from the automatic per-poll path; `AIRSPEED_ROLLING_KTS` and Rule 3/4 in `phase_detector.py` are unchanged.

### Bonus finding: a real, separate bug
While reproducing scenarios for this investigation, confirmed (via a real engine + phase detector, mocked audio) that **a session started airborne** (e.g. an MSFS "approach" start) fires "cleared for take-off" at touchdown during the landing rollout — the old automatic trigger had no way to distinguish "accelerating for takeoff" from "decelerating after landing, having started airborne." The manual-only redesign fixes this as a natural side effect, with no special-case code needed.

## Decision
**Implemented.** See [[ATC-Engine]] for the full mechanism detail and live-test status. `AIRSPEED_ROLLING_HYSTERESIS_KTS` (the B2 fix) is now flagged as a likely future cleanup item, since nothing dispatches off the speed boundary anymore — not yet removed, deliberately deferred to its own reviewed change.
