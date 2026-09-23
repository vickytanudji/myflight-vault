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

## Connection Robustness — ✅ Fully Fixed & LIVE-CONFIRMED (2026-09-23)

**Confirmed real, original bug:** `connect()`'s verification-read-returns-`None` branch dropped `self._sm` **without calling `exit()`** — leaks a handle + daemon dispatch thread per failed attempt. The leak needs a state where the handle opens but the sim doesn't answer (e.g. mid-load) — likely does NOT affect the main-menu scenario specifically, since MSFS keeps answering polls there.

**This went through a real "fixed → merged → still broken live" cycle, now resolved:**
1. First fix attempt (`_release_simconnect()` catching `AttributeError`/`OSError`) was merged and pytest-verified, but a live retest (2026-09-23) found it STILL threw `AttributeError: 'SimConnect' object has no attribute 'timerThread'` on every single MSFS-closed retry attempt (16+ consecutive).
2. **Root cause of why the first fix didn't work:** the wheel's `exit()` is `self.timerThread.join()` followed by `self.dll.Close(self.hSimConnect)` — the join line throws the `AttributeError` when `timerThread` was never set (because the wheel's internal `dll.Open()` can fail silently without setting it), so `dll.Close()` on the next line is **never reached**. The original fix's `except AttributeError` caught the symptom but the actual release action never ran — the handle stayed genuinely leaked, just silently instead of loudly.
3. **Real fix:** `_release_simconnect()` now checks for `timerThread` first; when absent, it calls `sm.dll.Close(sm.hSimConnect)` **directly**, bypassing `exit()` entirely (`dll`/`hSimConnect` are always set unconditionally in the wheel's `__init__`, before `connect()` ever runs, so this is always safe).
4. **Test gap that let the first broken fix merge undetected:** the old test fake had no `dll`/`hSimConnect` attributes at all, so no test could ever have asserted the real release action happened — it only asserted "exception caught, not propagated." Rebuilt to mirror the real wheel's `__init__` state, now asserts `dll.Close.assert_called_once_with(hSimConnect)` — the actual release, not just exception suppression.
5. **Live retest (2026-09-23): 3 consecutive MSFS-closed connection attempts, zero `timerThread` errors, fast clean failures (~0.44-0.47s) each time.** This is the confirmed real fix.

**Bonus fix, same brief:** console spam of raw `SIM def(b'GENERAL ENG COMBUSTION:1-4', b'Bool')` lines observed during the same test session, appearing while the flight-load gate polled with `CAMERA STATE` unreadable. Traced to the wheel's own internal `RequestList.py` retry-and-log behavior (not this project's code) — it uses stdlib `logging` (not loguru), specifically `Constants.py`'s logger (confirmed via object identity, due to wildcard-import shadowing), which falls to bare stderr with no prefix since nothing configures stdlib logging in this project. Now suppressed via `logging.getLogger("SimConnect.Constants").setLevel(logging.CRITICAL)` in `connect()`.

All identified gaps in this area are now closed, across three briefs:

1. **No timeout on the underlying wait — ✅ FIXED.** `python-SimConnect`'s `SimConnect()` constructor has an unbounded `while self.ok is False: pass` (confirmed directly against the pinned 0.4.26 wheel's source). A 10-second watchdog was added: on timeout, the wheel's own `ok` flag is set so the spin loop exits and the handle closes (no thread pile-up per retry, confirmed via a 3-timeouts-in-a-row test). Integrates cleanly with `reconnect_loop`'s existing retry. **One residual limit:** a worker stuck inside a native `Open` call can only be abandoned, not killed — closes its own handle if the call ever returns; never observed live.
2. **Adjacent leak — ✅ FIXED.** Failures at all three exit points (verification-read-None, verification-read-exception, `AircraftRequests()`/`Request()` raising) now route through one `_abandon_attempt()` helper, using `_release_simconnect()` internally.
3. **Silent swallow — ✅ FIXED.** `stop_polling()` now logs a WARNING on cleanup failure (kept the catch deliberately broad — narrowing it risks skipping `chartfox.close()`/`ws_server.stop()`, which run right after in `main.stop()`).
4. **Unguarded verification read — ✅ FIXED.** Now guarded, treated as an ordinary failed connection attempt; `reconnect_loop` survives a raising read.
5. **Event-loop stall — ✅ FIXED.** `reconnect_loop` now calls `await asyncio.to_thread(self.connect)` — a stalled attempt no longer blocks voice/WebSocket. Confirmed via a test that a concurrent coroutine keeps running during a stall.
6. **`stop_polling()`'s never-polled-handle leak — ✅ FIXED.** `if not self._running: ...; return` was skipping the entire handle-cleanup block, so `connect()` succeeding without a subsequent `start_polling()` left the handle open forever. Fixed via a new `_close_handle()` helper (same shared-helper pattern as `_abandon_attempt()`).
7. **`timerThread`-absent cleanup path (item 2 above) — ✅ FIXED AND LIVE-CONFIRMED**, the only one of these seven that went through a real merged-but-still-broken cycle before being genuinely fixed.

**Test coverage:** 102+ SimConnect tests, all passing, the majority written test-first (confirmed failing on unfixed code before each fix).

**Still outstanding (lower priority than the core connection bug, which is now confirmed):** a full `core.main` retest of the OTHER sub-fixes above (launch/close/reopen cycles, shutdown right after a successful connect before polling starts) — the one-liner script alone (which passed 3/3) doesn't exercise all of these paths.

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

**MSFS 2024 camera-state check for the flight-load gate is still outstanding** — never tested (2020 confirmed only, camera state `2` confirmed as "in cockpit").
