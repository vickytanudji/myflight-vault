---
tags: [testing, checklist, next-session]
---

# Next Test Session — Step by Step

This combines everything owed in [[09-Standing-Reminders]] into one session. Steps 1–2 are quick and isolated (do first, low risk). Steps 3–6 all happen across **one long flight** at YSSY, so plan for enough time to fly gate-to-gate. Step 7 is separate (needs real other traffic) — do it last, or in its own session if you're out of time.

---

## Step 1 — SimConnect connection-robustness check (do this before anything else loads)

This tests the connection layer itself, not the app's flight logic — do it cold, before you've loaded a flight.

1. **With MSFS fully closed**, run the one-liner from the connection-robustness brief:
   ```powershell
   python -c "import time; from core.simconnect.client import SimConnectClient as C; c=C(); t=time.perf_counter(); ok=c.connect(); print(ok, round(time.perf_counter()-t,3))"
   ```
   **Expect:** `False` returned quickly (not after a 10s hang).
2. **Launch MSFS**, wait for it to be fully up (main menu is enough), run the same one-liner again.
   **Expect:** `True`, and roughly `0.1` seconds.
3. Run `python -m core.main` **while MSFS is still closed** — confirm it doesn't hang, retries cleanly on its normal interval.
4. Now launch MSFS with `core.main` already running — confirm it connects once MSFS comes up.
5. **The specific new case to test:** get `core.main` running and connected, but **close MSFS again before loading a flight** (i.e. a shutdown/disconnect right after a successful connect, before polling ever really gets going). Confirm no crash, no hang, clean reconnect behavior afterward if you relaunch MSFS.

✅ Pass = no hangs anywhere, clean `False`→`True` transitions, no crash on the early-disconnect case.

---

## Step 2 — Flight-load gate quick check (MSFS 2020, cold and dark)

Skip this if you did it recently and nothing's changed — otherwise:

1. Load MSFS to the **main menu**. Confirm `core.main` shows no ATIS/ATC activity, just the "waiting for flight to load" log line repeating.
2. Load a flight **cold and dark** (not "ready to fly").
3. Confirm `CAMERA STATE` reaches `2` and the gate opens ~1.5s later, then ATIS speaks normally.

✅ Pass = silence at menu, normal startup once loaded.

---

## Step 3 — Enroute / Approach / Go-Around content fix (new this session)

Fly the full route. At each phase below, listen for the specific new content:

| Phase | Listen for |
|---|---|
| **Departure** (post-airborne) | Should hand off to Center — `"contact Center [freq]"` |
| **Enroute** | Should be a **cruise-altitude check-in**, NOT another Center handoff — e.g. `"[callsign], level [altitude]."` — confirm the Center handoff happened **once**, on Departure, not repeated here |
| **Approach** | Should be **one combined transmission**: descend instruction + approach clearance + Tower handoff, e.g. `"[callsign], descend and maintain [alt] feet, [type] approach runway [rwy] approved, contact Tower [freq]."` |
| **Go-Around** (if you can trigger one — climb away instead of landing) | First attempt should include a **real vectoring frequency**, not just "fly runway heading" with nothing else — e.g. `"...climb and maintain [alt] feet, fly runway heading, contact [freq] for vectors, expect vectors for another approach."` — confirm the frequency is a **real value**, not the synthetic default |

✅ Pass = each phase's new wording present, Departure/Enroute handoff not duplicated, Go-Around's frequency is real and audible.

---

## Step 4 — Frequency-tuning scenario (e): Center-sparse + Departure/Go-Around frequencies

Same flight as Step 3, `DISPATCH_TRIGGER_MODE=frequency_required`:

1. **At Enroute:** expect `"no real published CENTER frequency for YSSY... treated as satisfied"` — plays normally despite no real Center frequency existing.
2. **At Departure:** should require its own **real Departure frequency** (not Center) to unlock playback — try tuning the wrong frequency first, confirm it withholds, then tune correctly and confirm it plays.
3. **At Go-Around (if reached):** should gate on Tower or Approach's real frequency, not Center.

✅ Pass = Enroute auto-satisfies gracefully; Departure and Go-Around genuinely gate on their own real frequencies.

*(Scenarios a–d of this same checklist are already confirmed — see [[Frequency-Tuning-Retest-Checklist]] for full reference if anything looks off.)*

---

## Step 5 — MSFS 2024 camera-state check (only if you have time/access to 2024 today)

1. Launch **MSFS 2024** (not 2020) instead.
2. Load cold and dark as usual.
3. Watch the `CAMERA STATE changed: X -> Y` log lines as you go from menu to cockpit.
4. **Note down** whatever value appears once you're sitting in the cockpit, cold and dark.

This just needs recording — the flight-load gate already only checks for `{2, 3}`, which should work on both sims, but this confirms it for 2024 specifically.

---

## Step 6 — Wrap-up

Once through steps 1–5 (or as many as you get to), send me:
- Confirmation each step passed / what didn't
- Any log excerpts for anything unexpected
- The MSFS 2024 camera-state value from Step 5, if you did it

I'll update the vault and we'll figure out next steps from there.

---

## Step 7 — AI/FSLTL traffic probe (separate session — needs real other traffic)

**Do this on its own, not squeezed into the flight above.** Some scenarios need a VATSIM session or another human in multiplayer — see [[AI-FSLTL-Traffic]] for the full 21-item checklist before running `tools/probe_ai_traffic.py` for the first time live.
