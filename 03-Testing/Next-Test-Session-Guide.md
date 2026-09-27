---
tags: [testing, checklist, next-session]
---

# Next Test Session — Step by Step

## ✅ Already confirmed — no need to repeat these

- **SimConnect connection watchdog + `timerThread` cleanup fix** — confirmed live (2026-09-23), 3 consecutive clean MSFS-closed attempts, zero errors. Merged.
- **Flight-load gate** — confirmed reliably across every session since it was built. Camera state `2` opens the gate consistently on MSFS 2020.
- **Enroute/Approach/Go-Around content fix** — confirmed live across two separate flights. Enroute's cruise check-in, Approach's combined descend+clearance+Tower-handoff, and Go-Around's real vectoring frequency all sound correct.
- **APPROACH↔TOWER_ARRIVAL flapping fix + missed-go-around fix** — confirmed live across **two independent full flights**. Zero flapping on real descents (only 1-2 clean transitions each way, including one correctly-allowed genuine reversal at -1759fpm). Go-around correctly detected both via a direct `TOWER_ARRIVAL → GO_AROUND` transition and via the harder on-ground-blip case (`TOWER_ARRIVAL → TOWER_DEPARTURE → GO_AROUND`). Merging to `develop`.

---

## 🔲 Still to do

### Step 1 — Full `core.main` connection-robustness retest
The one-liner script (fixed and confirmed above) only tests the raw `connect()` call. Still owed: running the full app through a launch → close-MSFS → reopen-MSFS cycle, and specifically a shutdown right after a successful connect but *before* `start_polling()` starts — this exercises the never-polled-handle fix, which hasn't been exercised live yet.

### Step 2 — Frequency-tuning scenario (e): Center-sparse + Departure/Go-Around frequencies
Same flight as before, `DISPATCH_TRIGGER_MODE=frequency_required`:
1. **At Enroute:** expect `"no real published CENTER frequency for YSSY... treated as satisfied"` — plays normally despite no real Center frequency existing.
2. **At Departure:** should require its own **real Departure frequency** (not Center) to unlock playback — try tuning the wrong frequency first, confirm it withholds, then tune correctly and confirm it plays.
3. **At Go-Around (if reached):** should gate on Tower or Approach's real frequency, not Center.

✅ Pass = Enroute auto-satisfies gracefully; Departure and Go-Around genuinely gate on their own real frequencies.

*(Scenarios a–d of this same checklist are already confirmed — see [[Frequency-Tuning-Retest-Checklist]] for full reference if anything looks off.)*

### Step 3 — MSFS 2024 camera-state check (only if you have time/access to 2024 today)
1. Launch **MSFS 2024** (not 2020) instead.
2. Load cold and dark as usual.
3. Watch the `CAMERA STATE changed: X -> Y` log lines as you go from menu to cockpit.
4. **Note down** whatever value appears once you're sitting in the cockpit, cold and dark.

This just needs recording — the flight-load gate already only checks for `{2, 3}`, which should work on both sims, but this confirms it for 2024 specifically.

### Step 4 — AI/FSLTL traffic probe (separate session — needs real other traffic)
**Do this on its own, not squeezed into a normal flight.** Some scenarios need a VATSIM session or another human in multiplayer — see [[AI-FSLTL-Traffic]] for the full 21-item checklist before running `tools/probe_ai_traffic.py` for the first time live.

---

## Wrap-up
Once through any of the above, send the full log covering the relevant window, plus confirmation of what passed/failed, and the vault will get updated accordingly.
