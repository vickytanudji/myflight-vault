---
tags: [testing, checklist]
---

# Frequency-Tuning 5-Scenario Checklist

See [[Frequency-Tuned-Dispatch]] for architecture. Status of each scenario below.

## (a) Automatic mode regression — ✅ Confirmed
`DISPATCH_TRIGGER_MODE=automatic`. Fly through normally. **Expected:** no "withholding"/"frequency gate" log lines anywhere — identical to pre-feature behavior.

## (b) Wrong then right frequency — ✅ Confirmed
`DISPATCH_TRIGGER_MODE=frequency_required`.
1. Tune COM1 to a **wrong** frequency during Clearance.
2. **Expected:** `"...withholding playback until correct frequency is tuned"` — no audio.
3. Note the SID/squawk in the "transmission ready" log line.
4. Tune to the **correct** frequency (133.8 for Clearance Delivery at YSSY).
5. **Expected:** `"...playing the transmission generated earlier (replayed verbatim, not regenerated)"` — SID/squawk must match step 3.

## (c) Readback timing interaction — ✅ Confirmed
Once the withheld transmission actually plays: `"awaiting pilot readback"` must appear **only after** playback, never while withheld.

## (d) Manual trigger bypasses frequency gate — ✅ Confirmed
While on the wrong frequency:
```powershell
python -m tools.send_ws_request --type request_clearance
```
**Expected:** plays immediately regardless of frequency mismatch — explicit log line: `"manual request — staleness check and frequency gate bypassed"`.

## (e) Center-sparse fallback + Departure/Go-Around real frequencies — 🔲 STILL OWED
**Needs a longer flight** — past Ground/Tower Departure into Departure and Enroute.
- **Enroute:** expect `"no real published CENTER frequency for YSSY... treated as satisfied"` — plays normally despite no real frequency.
- **Departure:** should require its own **real** Departure frequency (not Center) — confirm it withholds/plays correctly against that.
- **Go-Around (if reached):** should gate on Tower or Approach's real frequency, not Center.

> [!todo] Do this on the next longer test flight — see [[09-Standing-Reminders]].
