---
tags: [architecture, frequency]
---

# Frequency-Tuned Dispatch

VATSIM-style trigger model: the pilot must have the correct real-world ATC frequency tuned on **COM1** to receive a phase's transmission. Built after the original power-state-based CLEARANCE trigger proved unreliable on some aircraft.

## Setting: `DISPATCH_TRIGGER_MODE`
- **`automatic`** (default) — **fully unchanged, byte-for-byte identical** to pre-existing behavior. Easy/casual mode. Confirmed via a hostile-gate-vs-no-gate regression test producing identical output.
- **`frequency_required`** — new realistic mode.

## How Gating Works
1. Transmission is **always generated and logged**, regardless of frequency.
2. Only **playback** is gated — mirrors the existing `_stale_current_phase` pattern in `ATCEngine`.
3. If pilot is on the wrong frequency: playback withheld, phase **NOT** marked issued (one-shot guard not consumed), logged clearly with both frequencies.
4. Once pilot retunes correctly: the **originally generated transmission replays verbatim** — never regenerated (would produce a different SID/squawk than what was logged as "ready," a real correctness bug this was designed to avoid).
5. Unreadable pilot frequency (0.0 / out of band) = **treated as mismatch**, not a pass (stricter than spec, deliberate).

## Real Frequency Sourcing (`core/atc/frequency_gate.py`)
- **Departure-side phases** (Clearance, Ground, Tower Departure): use `DepartureRunwayPlanner`'s GPS-resolved ICAO — **never** `current_airport_icao` directly (known unreliable, see [[SimConnect-Client]]).
- **Arrival-side phases** (Approach, Tower Arrival): reuse Approach's existing `_resolve_arrival_icao`.
- **Departure (post-airborne) / Go-Around:** gate on their *own* real controller (Departure freq; Tower/Approach respectively) — **not** Center. (Deviation from original brief, correctly reasoned: gating Departure on Center would shut out a pilot who tuned Departure exactly as Tower told them to.)
- **Enroute:** the one true Center-gated case. Since Center data is sparse/absent (see [[Center-Frequencies]]), Enroute auto-satisfies its own gate and plays normally when no real frequency exists.

## Manual Trigger Interaction
The manual `request_clearance` WebSocket trigger **bypasses the frequency gate** (and phase detection) by design — but does **not** bypass the flight-load gate or the one-shot guard.

## Setting Composition (ATC_TRIGGER_MODE × DISPATCH_TRIGGER_MODE, for Clearance)
| ATC_TRIGGER_MODE | DISPATCH_TRIGGER_MODE | Behavior |
|---|---|---|
| automatic | automatic | as before |
| ptt | automatic | as before (today's default) |
| automatic | frequency_required | generated once, held, plays once correct frequency tuned |
| ptt | frequency_required | needs right frequency **and** a PTT press; wrong frequency + PTT key → retune and key again, replays same stored clearance |

The other 7 phases follow `DISPATCH_TRIGGER_MODE` only (no PTT-priority concept for them).

## "Say Again" Interaction
Cannot release a withheld transmission — no readback entry exists yet for it, so pilot PTT input never reaches that logic while withheld. Once a transmission plays, readback/repeat handling is **not** frequency-gated (pilot may be monitoring on COM2).

## Live-Confirmed (MSFS 2020)
- ✅ (a) Automatic mode — pure regression, no gate lines appear
- ✅ (b) Wrong→right frequency — withheld, then verbatim replay confirmed by matching SID/squawk in logs
- ✅ (c) Readback-gating timing — `awaiting readback` only appears after actual playback
- ✅ (d) Manual trigger bypasses frequency gate — confirmed via explicit log line
- 🔲 (e) Center-sparse fallback + Departure/Go-Around real-frequency requirement — **deferred to a longer flight**

## Known, Permanent Limitation
Center/ARTCC frequency will (almost) always be synthetic. See [[Center-Frequencies]]. This is accepted, not a bug.

## COM1 Reliability (confirmed via probe)
`tools/probe_com1_frequency.py` confirmed COM1 **and** COM2 read reliably across two very different aircraft (Cessna 172, Boeing 787) — clean response to real retuning, no dropouts, no frozen values. COM2 itself is **out of scope for V1** but the read path would extend cleanly.
