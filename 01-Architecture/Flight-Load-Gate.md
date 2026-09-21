---
tags: [architecture, startup]
---

# Flight-Load Gate (`core/atc/flight_load_gate.py`)

Prevents the **entire ATC/ATIS pipeline** from starting until a real flight is actually loaded — not merely SimConnect-connected. Built after discovering the app would fully run (ATIS, phase detection, CLEARANCE dispatch) while sitting at the MSFS main menu with only default/placeholder telemetry.

## Why This Was Needed
- `FlightData.is_stale()` only proves a poll happened — MSFS keeps answering SimConnect at the main menu too, so staleness checks can't distinguish menu from a real flight.
- Telemetry itself can't distinguish them either — a cold-and-dark aircraft on the ramp looks identical to menu placeholder values (near-zero altitude, on_ground=True, engines off).
- Waiting for data to *vary* also fails for the same reason.

## The Real Signal: `CAMERA STATE`
Read via raw `SimConnect.Request` (missing from the `AircraftRequests` wrapper table, same pattern as `EXTERNAL POWER ON:1`).

> [!warning] Enum values differ between MSFS 2020 and 2024
> E.g. value `9` = "Showcase" in 2020 but "Waiting" in 2024. **Do not build a menu-value list.** The gate opens only on the two values that mean the same thing in both sims: **`{2 (Cockpit), 3 (External/Chase)}`**.

**Confirmed live (MSFS 2020):** camera state `15` = main menu specifically. State `2` = in cockpit — confirmed as the correct open-trigger value; gate opened ~4 seconds after reaching it (3 consecutive confirming polls, ~0.5–1s each).

## Gate Logic
- Opens after **3 consecutive polls** (~1.5s) with: fresh data, camera state ∈ {2, 3}, and a real (non-0,0) position.
- Once open, **stays open for the rest of the run.**
- Until open: ATIS silent, phase detector not fed (would otherwise baseline on menu placeholder altitude — reproduced rejecting 10 real polls at a 5434ft airport), no ATC dispatch, PTT dropped before transcription. Main loop keeps polling/broadcasting internally.
- **Fails closed** if camera state is unreadable — deliberate choice over silently falling back to old (buggy) behavior. Logs how to bypass (`FLIGHT_LOAD_GATE_ENABLED=false` kill switch).

## Config
- `FLIGHT_LOAD_GATE_ENABLED` — kill switch, restores old behavior entirely (including for the manual trigger)
- `FLIGHT_LOADED_CAMERA_STATES` — editable in `.env` without a code change, in case future testing finds a value outside {2, 3} also means "loaded"

## Manual Trigger Interaction (fixed after initial oversight)
The manual `request_clearance` trigger originally only checked staleness, which would have let it fire at the main menu. **Now additionally requires the gate to be open** — additive with the staleness check, not a replacement. A rejection while gate-closed does **not** consume the one-shot guard (confirmed live — same request worked once the flight loaded).

## Live-Confirmed (MSFS 2020)
- ✅ No ATIS/ATC activity at main menu (60–90+ seconds observed)
- ✅ Manual trigger correctly rejected at menu with clear log line
- ✅ Gate opens correctly on reaching cockpit (camera state 2)
- ✅ Manual trigger works normally once loaded

## Still Unknown
- What camera state a cold-and-dark **MSFS 2024** load reports (2020 confirmed only)
- Whether "Ready to Fly" reports state 2 *before* pressing "Fly" (not deliberately tested — real-world usage tonight didn't show it opening prematurely, treated as good-enough confirmation)
