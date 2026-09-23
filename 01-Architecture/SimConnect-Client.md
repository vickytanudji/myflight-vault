---
tags: [architecture, simconnect]
---

# SimConnect Client (`core/simconnect/client.py`)

## The `AircraftRequests` Wrapper Gap Pattern
`python-SimConnect==0.4.26`'s `AircraftRequests.get()` wrapper only resolves names in its own internal static table (`RequestList.py`). **A miss returns `None` silently — does not mean the SimVar doesn't exist in MSFS.** Recurring pattern this project has hit multiple times:

| SimVar | In wrapper table? | Fix |
|---|---|---|
| `ATC RUNWAY AIRPORT NAME` | ❌ No | Raw `SimConnect.Request` bypass |
| `EXTERNAL POWER ON:1` | ❌ No | Raw `SimConnect.Request` bypass |
| `CAMERA STATE` | ❌ No | Raw `SimConnect.Request` bypass |
| `COM_ACTIVE_FREQUENCY:1/2` | ✅ Yes | Normal wrapper path — confirmed reliable across 2 aircraft |

## `ATC RUNWAY AIRPORT NAME` — Display Name, Not ICAO Code (major fix this session)
**The bug:** this field returns a human-readable airport **name** ("Kingsford Smith Intl"), never an ICAO code. Neither the 2020 nor 2024 SDK docs ever claimed otherwise — the "ICAO code" assumption originated from a wrong label in `docs/sdk/simconnect-variables.md` (now corrected).

**Confirmed:** present on **both MSFS 2020 and 2024** — not version-specific. Was silently active this entire project's history.

**What broke:** because every consumer only checked "is this non-empty," a display name did *worse* than an empty string — it **skipped** the `DEFAULT_DESTINATION_ICAO` fallback entirely and got sent to ChartFox / frequency lookups as if it were a real code. Affected: Approach (`_resolve_arrival_icao`), the arrival-side frequency gate, Go-Around (same resolver), ATIS, Departure/Enroute Center lookups (fell back to default anyway, so lower impact there), and `main.py`'s pilot-transmission handler.

**Fix:** validated at the source in `client.py` — a value passes only if it is **exactly 4 uppercase letters**, else `current_airport_icao` becomes `""` (the existing "unavailable" signal every consumer already handles). No downstream consumer was patched individually.

**Consequence:** `current_airport_icao` is now `""` almost always. **Don't build new logic assuming it will populate.** Use `core.atc.runway_selection.resolve_departure_icao` (GPS-based) for a real airport instead.

**Not salvageable for ICAO resolution** — the raw value was never ICAO-shaped in anything found, and deriving a code from the name isn't safe (airportsdata's names don't match SimConnect's display names, and name collisions exist across ~973 airports).

<<<<<<< HEAD
## Connection Robustness — ⚠️ REGRESSION FOUND (live retest, 2026-09-23)

**Confirmed real, original bug:** `connect()`'s verification-read-returns-`None` branch dropped `self._sm` **without calling `exit()`** — leaks a handle + daemon dispatch thread per failed attempt. ~12 leaks/minute if hit repeatedly (5s retry interval) — but likely does NOT affect the main-menu scenario specifically, since MSFS keeps answering polls there and `connect()` succeeds without retrying. The leak needs a state where the handle opens but the sim doesn't answer (e.g. mid-load).

**⚠️ LIVE RETEST (2026-09-23) FOUND THE FIX ITSELF IS STILL BROKEN.** With MSFS closed, running `core.main` (or the one-liner script) reproducibly hits this on EVERY SINGLE retry attempt (confirmed across 16+ consecutive retries in one session):
```
WARNING core.simconnect.client:927 - SimConnect cleanup after failed connect raised 
AttributeError: 'SimConnect' object has no attribute 'timerThread' - the handle may not have been released
```
This is the EXACT AttributeError the original leak investigation identified and was supposed to fix (`_release_simconnect()` was meant to catch this). The watchdog timeout itself IS working correctly (fails fast at ~0.4-0.5s, not a 10s hang) — but the cleanup call that runs after that fast failure is itself throwing, and per the warning text, "the handle may not have been released," meaning the original leak may not actually be fixed at all, just newly detected and logged instead of silently happening.
**Confirmed on latest `develop`, post-pull, not a stale-checkout issue.**

**Also observed live, same test session:** console spam of raw `SIM def(b'GENERAL ENG COMBUSTION:1-4', b'Bool')` lines, repeating in tight clusters specifically while the flight-load gate is polling with `CAMERA STATE` unreadable. Not seen in earlier sessions' logs — possibly the polling loop is redefining these SimVar requests every poll cycle instead of once at startup. Unclear if related to the cleanup bug or a separate issue. Needs investigation.

All identified gaps in this area were believed closed, across two briefs — **the live retest shows at least the original leak-fix (item 2 below) did not actually work as intended:**
=======
## Connection Robustness — ✅ Fully Fixed & Merged (this session)

**Confirmed real, original bug:** `connect()`'s verification-read-returns-`None` branch dropped `self._sm` **without calling `exit()`** — leaks a handle + daemon dispatch thread per failed attempt. ~12 leaks/minute if hit repeatedly (5s retry interval) — but likely does NOT affect the main-menu scenario specifically, since MSFS keeps answering polls there and `connect()` succeeds without retrying. The leak needs a state where the handle opens but the sim doesn't answer (e.g. mid-load).

All identified gaps in this area are now closed, across two briefs:
>>>>>>> origin/main

1. **No timeout on the underlying wait — ✅ FIXED.** `python-SimConnect`'s `SimConnect()` constructor has an unbounded `while self.ok is False: pass` (confirmed directly against the pinned 0.4.26 wheel's source). A 10-second watchdog was added: on timeout, the wheel's own `ok` flag is set so the spin loop exits and the handle closes (no thread pile-up per retry, confirmed via a 3-timeouts-in-a-row test). Integrates cleanly with `reconnect_loop`'s existing retry. **One residual limit:** a worker stuck inside a native `Open` call can only be abandoned, not killed — closes its own handle if the call ever returns; never observed live.
2. **Adjacent leak — ✅ FIXED.** Failures at all three exit points (verification-read-None, verification-read-exception, `AircraftRequests()`/`Request()` raising) now route through one `_abandon_attempt()` helper, using `_release_simconnect()` internally (catches both `AttributeError` and `OSError` — the wheel's `SimConnect_Close` has `restype=HRESULT`, so a failed close can raise `OSError`, which would otherwise escape and kill `reconnect_loop`).
3. **Silent swallow — ✅ FIXED.** `stop_polling()` now logs a WARNING on cleanup failure (kept the catch deliberately broad — narrowing it risks skipping `chartfox.close()`/`ws_server.stop()`, which run right after in `main.stop()`).
4. **Unguarded verification read — ✅ FIXED.** Now guarded, treated as an ordinary failed connection attempt; `reconnect_loop` survives a raising read.
5. **Event-loop stall — ✅ FIXED.** `reconnect_loop` now calls `await asyncio.to_thread(self.connect)` — a stalled attempt no longer blocks voice/WebSocket. Confirmed via a test that a concurrent coroutine keeps running during a stall.
6. **`stop_polling()`'s never-polled-handle leak — ✅ FIXED (most recent brief).** `if not self._running: ...; return` was skipping the entire handle-cleanup block, so `connect()` succeeding without a subsequent `start_polling()` left the handle open forever. Fixed via a new `_close_handle()` helper (same shared-helper pattern as `_abandon_attempt()`), while keeping the thread-join logic correctly gated on `self._running` for the normal path. Deliberately kept the *broader* exception catch here too, for the same `main.stop()` reason as item 3.

**Test coverage:** 97 SimConnect tests (post-robustness-brief) → 102 tests (post-never-polled-handle fix), all passing, ~32 written test-first (confirmed failing on unfixed code before the fix).

**⚠️ MSFS 2020 live retest of ALL of the above is still outstanding** — nothing in this section has run against a real sim yet. When testing: run the provided one-liner script, test with MSFS closed, launch MSFS, close/reopen it, and specifically test a shutdown right after a successful connect (before `start_polling()` runs) to exercise the newest fix.

**Residual, not-yet-fixed, low-priority:** if `reconnect_loop` is cancelled mid-attempt, the worker thread still finishes (up to 10s) before shutdown completes — same worst case as the old fully-blocking behavior, not worse, just not eliminated.

## Confirmed-Fixed Behaviors — DO NOT REVERT
| Field | Correct behavior |
|---|---|
| `VERTICAL_SPEED` | Used as-is — no ×60 conversion (wrapper requests fpm already) |
| `GROUND_VELOCITY` | Wrapper requests **Knots** directly — comment previously said "feet/second," now corrected (no actual wrong conversion existed in code) |
| Latitude/Longitude | Used as-is — no `math.degrees()` conversion |
| `COM1` active frequency | Plain float — no BCD16 decode |
| `current_airport_icao` | Raw `SimConnect.Request`, validated as 4-uppercase-letters or `""` |

## MSFS 2024 Compatibility
See [[MSFS-2024-Support]]. All 25+ SimVars this project reads are **confirmed identical** in value/unit/semantics between 2020 and 2024, live-tested. Same pinned wheel works unmodified. No sim-version detection/branching needed anywhere in the codebase.

**Dual-support formalized** (`feat/formalize-dual-sim-support`, merged): `CLAUDE.md` now has a "Supported simulators" section; `docs/sdk/simconnect-variables.md` has detailed variance notes. Three real, documented differences between 2020/2024 (none requiring code changes):
1. AI-traffic state behaves differently (relevant to [[AI-FSLTL-Traffic]] later)
2. Some default 2024 aircraft have external-power/APU telemetry issues — same class already handled by `POWER_TELEMETRY_FALLBACK_POLLS`
3. **`CAMERA STATE` enum numbering differs between sim versions** (e.g. `9` = "Showcase" in 2020, "Waiting" in 2024) — this is why [[Flight-Load-Gate]] deliberately only checks the two values that mean the same in both sims (`{2, 3}`), not a menu-value list.

**MSFS 2020 live retest of the watchdog fix itself is still outstanding** — bundled with the broader connection-robustness retest above.
