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

## Rule Table (Rules 1–3 redesigned earlier this project; Rules 7–8's boundary since hardened, see below)
| Rule | Condition | Result |
|---|---|---|
| 0 | current phase ∈ GO_AROUND_ELIGIBLE, airborne, vs > climb threshold | hold current phase |
| 1 | on ground, no engines, no ground/APU power | PARKED |
| 2 | on ground, no engines, ground/APU power active (debounced) | CLEARANCE |
| 3 | on ground, engines running | GROUND |
| 4 | on ground, ias ≥ 40kt | TOWER_DEPARTURE |
| 5–6 | airborne rules (unchanged) | DEPARTURE / ENROUTE |
| 7 | airborne, vs≤-500fpm | APPROACH |
| 8 | airborne, vs≤-200fpm, alt<2500ft | TOWER_ARRIVAL |

## APPROACH ↔ TOWER_ARRIVAL Boundary Hardening (2026-09-25) — ✅ fixed, live-confirmed twice

Rule 7's altitude condition is a superset of Rule 8's, so TOWER_ARRIVAL can only ever be reached in the narrow (-500,-200] vs band — and real descent vertical speed genuinely, repeatedly oscillates across -500fpm from ordinary control/sensor noise. Confirmed via two real flights with meaningfully different noise widths (~-401 to -661 on one; ~-327 to -933 on another).

**A first fix (a flat +200fpm hysteresis margin, reverting to APPROACH only below -700fpm) was insufficient** — tuned against one sample, failed live against a wider second one. The real fix combines:
- **`VS_BOUNDARY_HYSTERESIS_FPM = 600.0`** — revert-to-APPROACH threshold is now -1100fpm, not -500fpm
- **`VS_BOUNDARY_REVERSAL_DEBOUNCE_POLLS = 4`** — the reversal-worthy vs value must hold for 4 consecutive polls (smaller than `GO_AROUND_CONFIRM_POLLS`'s 5, larger than the generic 2-poll debounce already proven insufficient here)

Reasoning: a transient noise spike is short-lived; a genuine reversal (real climb/level-off) is sustained by definition. Combining a magnitude check with a duration check generalizes better to an unseen third approach than scaling magnitude alone — the required margin had already grown ~2.5x between just two samples with no evident ceiling, so a bigger fixed number alone was the same failed reasoning at larger scale.

**Live-confirmed twice, zero flapping on real descents.** Notably, one genuine sustained reversal (-1759fpm, a real level-off) was correctly let through as a real transition rather than blocked — confirms the fix doesn't over-suppress legitimate reversals.

## Missed Go-Around via a Brief On-Ground Blip (2026-09-25) — ✅ fixed, live-confirmed twice

A real go-around during a low pass can produce a single `on_ground=True` poll at very low altitude, high ground speed, still-negative vertical speed — internally consistent enough to pass the speed-divergence gate (only a 30kt threshold) and the altitude-continuity check (deliberately skipped on any on-ground flip, to avoid the historic frozen-baseline bug). Per the rule table, this single poll matches Rule 4 (TOWER_DEPARTURE), which is NOT in `GO_AROUND_ELIGIBLE_PHASES` — so the subsequent climb was read as an ordinary takeoff instead of a go-around.

**Investigated first: neither existing gate was ever meant to catch this** — both work exactly as designed for their own purposes; this was a genuinely new real-world case, not a bug in an existing mechanism.

**Fix:** new `_go_around_eligible()` helper — `TOWER_DEPARTURE` is treated as go-around-eligible too, specifically when the immediately-preceding confirmed phase was APPROACH/TOWER_ARRIVAL/GO_AROUND. `GO_AROUND_ELIGIBLE_PHASES` itself is untouched. Real takeoffs (always preceded by GROUND) and real landings (never leave `on_ground=True`) are structurally excluded from the change, not just assumed safe.

**Live-confirmed twice** — the easy case (a go-around that stays airborne throughout, direct `TOWER_ARRIVAL → GO_AROUND`) and the hard blip case (`TOWER_ARRIVAL → TOWER_DEPARTURE → GO_AROUND`) both confirmed working on real flights.

## See Also
- [[Flight-Load-Gate]] — gates *when* the detector starts being fed real data at all
- [[07-Bug-Log]] — full history of bugs found in this file
