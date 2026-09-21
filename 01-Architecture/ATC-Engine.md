---
tags: [architecture, engine]
---

# ATC Engine (`core/atc/engine.py`)

Generic phase-dispatch engine. Each `FlightPhase` (except PARKED) is `register()`-ed with a handler coroutine in `main.py`. `handle_phase(phase, flight_data)` looks up and calls the handler; if none registered, silent no-op (this silence is exactly how the historic Tower Arrival bug went undetected for so long).

## One-Shot Guard
`_issued_phases` tracks which phases have fired this session. A phase fires **once** unless re-armed (Go-Around re-arms Approach/Tower Arrival on confirmation).

## Templated Generation (this session's major refactor)
All 9 phases now render from **fixed ICAO templates** (`core/atc/phraseology/icao.py`, `render_*_template()`) instead of calling the LLM for the normal case. See `docs/phraseology_reference.md` for the frozen template text and citation confidence per phase.

### FREE_RESPONSE Fallback
Each phase has a `free_response_reason()` check, evaluated before templating:
- **Category (a) — currently triggerable:** a required field resolves to None/empty with no template branch (rare — e.g. totally unresolvable callsign)
- **Category (b) — future hooks, always False today:** traffic advisory needed, non-standard instruction, holding/weather deviation, emergency phraseology, readback correction needed

If triggered, falls back to the original LLM-based generation (kept intact, never deleted) and logs a WARNING.

## Frequency Gating (this session, new)
See [[Frequency-Tuned-Dispatch]] for full detail. Summary: transmission is **always generated and logged**; only **playback** is gated on the pilot being tuned to the correct real frequency. Mirrors the existing `_stale_current_phase` pattern.

## Flight-Load Gating (this session, new)
See [[Flight-Load-Gate]]. Nothing dispatches until a real flight is confirmed loaded (via `CAMERA STATE`), not just SimConnect-connected.

## Readback-Gating
See [[Readback-Gating]] — a separate, additive layer on top of dispatch: after a qualifying transmission plays, the engine waits for a PTT-confirmed readback before considering the interaction complete.

## Manual "Request Clearance" Trigger (this session, new)
- Inbound WebSocket message `{"type": "request_clearance"}` → directly dispatches CLEARANCE
- **Bypasses:** phase detection, frequency gate
- **Does NOT bypass:** the flight-load gate (fixed after initial oversight), or the one-shot guard
- Test harness: `python -m tools.send_ws_request --type request_clearance`

## Known Un-Fixed Robustness Gaps (found, not yet fixed as of last session)
- `stop_polling()`'s `except Exception: pass` around exit() swallows failures silently
- Same handle-leak class as the fixed one, but in the *first* except branch (if `AircraftRequests()`/`Request()` raises after `SimConnect()` succeeds)
- The verification read itself is unguarded — an exception there bypasses cleanup and kills `reconnect_loop`

> These three are all in [[SimConnect-Client]]'s `connect()` — same bug class as the fixed leak, different trigger points. Worth a follow-up brief.
