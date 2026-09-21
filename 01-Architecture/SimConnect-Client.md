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

## Connection Handle Leak (found & partially fixed this session)
**Confirmed real:** `connect()`'s verification-read-returns-`None` branch dropped `self._sm` **without calling `exit()`** — leaks a handle + daemon dispatch thread per failed attempt.
- **Fixed:** now calls a `_release_simconnect()` helper on this path, catching both `AttributeError` *and* `OSError` (the wheel's `SimConnect_Close` has `restype=HRESULT`, so a failed close can raise `OSError` — this would otherwise escape and kill `reconnect_loop`).
- **Practical impact assessed:** ~12 leaks/minute if this state is hit repeatedly (5s retry interval) — but likely **does NOT affect the main-menu scenario** specifically, since MSFS keeps answering polls there and `connect()` succeeds without retrying. The leak needs a state where the handle opens but the sim doesn't answer (e.g. mid-load).

### Still Open (found, not yet fixed)
1. `stop_polling()`'s `except Exception: pass` — silently swallows cleanup failures
2. **Same leak class, different branch:** if `SimConnect()` succeeds but `AircraftRequests()`/`Request()` then raises, `self._sm` is dropped without `exit()` too
3. **Unguarded verification read** — an exception there (not just a `None` return) bypasses cleanup entirely and kills `reconnect_loop`
4. **No timeout on the underlying wait** — ✅ **FIXED, merged.** `python-SimConnect`'s `SimConnect()` constructor has an unbounded `while self.ok is False: pass` (confirmed directly against the pinned 0.4.26 wheel's source). A 10-second watchdog was added: on timeout, the wheel's own `ok` flag is set so the spin loop exits and the handle closes (no thread pile-up per retry, confirmed via a 3-timeouts-in-a-row test). Integrates cleanly with `reconnect_loop`'s existing retry. **One residual limit:** a worker stuck inside a native `Open` call can only be abandoned, not killed — closes its own handle if the call ever returns; never observed live.
   - **Adjacent leak — ✅ FIXED, merged.** Failures at all three exit points (verification-read-None, verification-read-exception, `AircraftRequests()`/`Request()` raising) now route through one `_abandon_attempt()` helper.
   - **Silent swallow — ✅ FIXED, merged.** `stop_polling()` now logs a WARNING on cleanup failure (kept the catch deliberately broad — narrowing it risks skipping `chartfox.close()`/`ws_server.stop()`, which run right after in `main.stop()`).
   - **Event-loop stall — ✅ FIXED, merged.** `reconnect_loop` now calls `await asyncio.to_thread(self.connect)` — a stalled attempt no longer blocks voice/WebSocket. Confirmed via a test that a concurrent coroutine keeps running during a stall.
   - **All 97 SimConnect tests pass, 17 new ones written test-first (15 failed on unfixed code).**
   - **⚠️ MSFS 2020 live retest of all of the above is still outstanding** — nothing here has run against a real sim yet.
   - **New, still-open items found during this work:** `stop_polling()`'s early-return means a connected-but-never-polled handle isn't closed on shutdown (different leak scenario); a cancelled `reconnect_loop` attempt still waits up to 10s for its worker thread on shutdown (same worst case as before, not worse, just not eliminated).

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

**Dual-support formalized** (`feat/formalize-dual-sim-support`, committed): `CLAUDE.md` now has a "Supported simulators" section; `docs/sdk/simconnect-variables.md` has detailed variance notes. Three real, documented differences between 2020/2024 (none requiring code changes):
1. AI-traffic state behaves differently (relevant to [[AI-FSLTL-Traffic]] later)
2. Some default 2024 aircraft have external-power/APU telemetry issues — same class already handled by `POWER_TELEMETRY_FALLBACK_POLLS`
3. **`CAMERA STATE` enum numbering differs between sim versions** (e.g. `9` = "Showcase" in 2020, "Waiting" in 2024) — this is why [[Flight-Load-Gate]] deliberately only checks the two values that mean the same in both sims (`{2, 3}`), not a menu-value list.

**MSFS 2020 live retest of the watchdog fix itself is still outstanding** — code is committed, not yet run against a real sim.
