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

## Manual "Request Clearance" Trigger
- Inbound WebSocket message `{"type": "request_clearance"}` → directly dispatches CLEARANCE
- **Bypasses:** phase detection, frequency gate
- **Does NOT bypass:** the flight-load gate (fixed after initial oversight), or the one-shot guard
- Test harness: `python -m tools.send_ws_request --type request_clearance`

## Manual Takeoff Clearance (2026-09-27) — ✅ implemented, Tests 1&2 live-confirmed

**Replaces automatic speed-based dispatch entirely.** Previously, the takeoff-clearance transmission fired automatically whenever the phase detector reported `TOWER_DEPARTURE` (a raw 40kt IAS threshold, per [[Phase-Detector]]'s Rule 3/4). This had two real problems: (1) a residual flapping risk on a windy fast taxi-out could still fire the clearance prematurely mid-taxi (documented, accepted trade-off from the B2 hysteresis fix), and (2) a session started **airborne** (e.g. an MSFS "approach" start) would fire "cleared for take-off" at touchdown during the landing rollout — a confirmed, real bug, reproduced with a real engine + phase detector (mocked audio) before the fix.

**Design (from `docs/investigations/position-based-takeoff-clearance.md`):** investigated `ON_ANY_RUNWAY` as a possible position-based signal first — confirmed unreliable (missing from the pinned SimConnect wrapper, plus two open MSFS DevSupport bug reports on both 2020 and the 2024 SU4 beta showing it reads False even while genuinely on a runway). Correctly stopped at that investigation gate rather than building on an unconfirmed SimVar. Recommended and implemented **manual-only** instead:

- **New WebSocket trigger, `request_takeoff_clearance`** — mirrors `request_clearance`'s shape. Skips the frequency check and the "aircraft has moved on" staleness check, but NOT the one-shot guard (a second request is a no-op, logged, guard not consumed). Refused (with a warning, guard not consumed) if: flight data is stale, no flight is loaded, not `on_ground`, no engine running, or the phase isn't GROUND or TOWER_DEPARTURE.
- **New PTT voice phrase trigger** — "ready for departure", "ready for immediate departure", "ready for takeoff"/"take off", "request(ing) takeoff"/"take off". Deliberately does NOT match a bare "takeoff" alone, nor the pilot's own readback phrasing ("cleared for takeoff", "request departure clearance") — confirmed live: bare "Take off." correctly did not trigger.
- **Ordering:** the phrase-intent check runs BEFORE readback routing — otherwise "ready for departure" while Ground's readback is still outstanding would get swallowed as a malformed readback attempt instead of recognized as a departure request.
- **Phase requirement (GROUND or TOWER_DEPARTURE, either one):** deliberately broader than a single phase, to avoid depending on the noisy 40kt boundary between them for ELIGIBILITY (only for the phase *label*, which still updates normally and is still needed elsewhere — go-around graze detection, stale-transmission ordering — untouched by this fix).
- **Known, documented, accepted gap:** in `frequency_required` mode, a manual takeoff request bypasses the frequency check, so a pilot could hear "cleared for take-off" before Ground's own (still-withheld, wrong-frequency) taxi clearance ever plays. Narrow edge case, not yet fixed — a pilot manually requesting takeoff implies they're already effectively at the runway, so the missed taxi clearance is largely moot by that point.
- Test harness: `tools/send_ws_request.py --takeoff`.

**Live-confirmed (2026-09-27):** WebSocket trigger fires correctly and exactly once (Test 1); voice phrase trigger fires correctly, and the one-shot guard correctly blocked a second attempt via a DIFFERENT trigger path than the first (Test 2 — confirms guard sharing across both trigger mechanisms). Bare "take off" and STT-garbled "ready for the patcher" both correctly did NOT trigger.
**Still owed (deferred, not urgent):** Test 3 (confirm the airborne-start touchdown bug is actually fixed live, not just via the mocked regression test) and Test 4 (sanity-check go-around/Ground-readback interactions are unaffected).

**Future cleanup flagged, not yet done:** `AIRSPEED_ROLLING_HYSTERESIS_KTS` (the B2 phase-detector fix) is now likely unneeded for dispatch purposes — it only steadies the phase *label* during landing rollout now, since nothing dispatches off it anymore. Not removed yet; should get its own reviewed change.

## SimConnect Connection Robustness — ✅ All Fixed & Merged
The four robustness gaps once tracked here (adjacent handle leak, silent `stop_polling()` swallow, unguarded verification read, event-loop stall on `reconnect_loop`) are **all fixed and merged**. Full detail in [[SimConnect-Client]]. **MSFS 2020 live retest of these fixes is still outstanding.**

Three smaller follow-up items surfaced during that work and are queued in a new brief (not yet run) — see [[07-Bug-Log]] and [[09-Standing-Reminders]]:
- `handle_pilot_transmission` returning silence (`""`) instead of a fallback phrase on LLM failure
- `_run_or_skip`'s test-skip logic being broader than ideal
- `stop_polling()`'s early-return not closing a connected-but-never-polled handle
