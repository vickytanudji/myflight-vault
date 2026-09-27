---
tags: [testing, checklist, next-session]
---

# Next Test Session — Step by Step

## ✅ Already confirmed — no need to repeat these

- **SimConnect connection watchdog + `timerThread` cleanup fix** — confirmed live (2026-09-23), 3 consecutive clean MSFS-closed attempts, zero errors. Merged.
- **Flight-load gate** — confirmed reliably across every session since it was built. Camera state `2` opens the gate consistently on MSFS 2020.
- **Enroute/Approach/Go-Around content fix** — confirmed live across two separate flights. Enroute's cruise check-in, Approach's combined descend+clearance+Tower-handoff, and Go-Around's real vectoring frequency all sound correct.
- **APPROACH↔TOWER_ARRIVAL flapping fix + missed-go-around fix** — confirmed live across **two independent full flights**. Zero flapping on real descents (only 1-2 clean transitions each way, including one correctly-allowed genuine reversal at -1759fpm). Go-around correctly detected both via a direct `TOWER_ARRIVAL → GO_AROUND` transition and via the harder on-ground-blip case (`TOWER_ARRIVAL → TOWER_DEPARTURE → GO_AROUND`). Merged to `develop`.
- **MSFS 2024 cold-and-dark camera-state check (2026-09-27)** — confirmed: reads `CAMERA STATE = 2`, same as MSFS 2020's cockpit value. Real transition sequence observed (`12 → 35 → 32 → 30 → 16 → 2`) confirms 2024's intermediate loading states genuinely differ from 2020's, but the gate's `{2, 3}` trigger values work correctly unmodified on both sims.
- **Readback matching (frequency format, misheard "decimal", callsign format) + stale-readback cleanup + manual takeoff-clearance redesign (2026-09-27)** — all confirmed live. See [[07-Bug-Log]] for full detail on each.

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

### Step 3 — AI/FSLTL traffic probe (separate session — needs real other traffic)
**Do this on its own, not squeezed into a normal flight.** A minimal, default-OFF, passive/logging-only V1 has been implemented (`TRAFFIC_AWARENESS_ENABLED=true`, not yet reviewed/merged) — needs a real AI traffic density in MSFS to get the first-ever real data point. Some scenarios need a VATSIM session or another human in multiplayer — see [[AI-FSLTL-Traffic]] for the full 21-item checklist before running the live check.

---

## Wrap-up
Once through any of the above, send the full log covering the relevant window, plus confirmation of what passed/failed, and the vault will get updated accordingly.
