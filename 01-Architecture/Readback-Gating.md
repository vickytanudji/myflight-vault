---
tags: [architecture, readback]
---

# Readback-Gating Subsystem

A PTT-based confirmation step required after specific ATC transmissions, per real ICAO/Annex 11 §3.7.3.1 readback rules. **Separate mechanism from dispatch** — transmissions still fire automatically (or per frequency-gate rules); readback-gating governs what happens *after* a transmission plays.

Qualifying phases: Clearance, Ground, Tower Departure, Departure, Enroute, Approach, Tower Arrival, Go-Around. **ATIS does not require readback** (informational broadcast).

## Setting: `READBACK_VALIDATION_LEVEL`

### Level 1 — `presence_only`
Any non-empty PTT-captured transcription satisfies the readback, content ignored. ✅ Confirmed working live, zero bugs found.

### Level 2 — `keyword_match`
Readback must contain key expected elements (callsign, runway, altitude, squawk, SID name, destination as applicable) — no strict phraseology required.

**3 real bugs found & fixed via live testing:**
1. **Digit/fuzzy matching** — STT renders numbers many ways ("842" / "eight forty-two" / "8-4-2"); SID names get mis-transcribed ("Fisha" → "Fischer"). Fixed with digit normalization + fuzzy SID/destination matching.
2. **Runway-span parsing** — "3, 4, right" / "three four right" wasn't recognized as matching "34R". Fixed with a dedicated inverse-of-`spoken_runway()` parser, bounded to real ICAO ranges (01–36), never accepting an arbitrary parsed runway — only one matching the *specific expected* value (format tolerance, not correctness tolerance).
3. **Multi-pending routing** — with 2 readbacks pending simultaneously (e.g. Clearance + Ground), an incoming response was routed to the wrong one using a naive "fewest missing elements" heuristic. Fixed with content-signal-based routing (taxi phraseology → Ground-family; SID/squawk-shaped content → Clearance) before falling back to the missing-count heuristic.

### Level 3 — `full_icao_phraseology`
Full structural validation + correction loop.
- Failed readback → ATC says **"[callsign], readback incorrect, say again"** → returns to awaiting state
- `MAX_READBACK_ATTEMPTS = 3` — after 3 failures, gives up gracefully, logs a WARNING, marks phase complete without valid readback (does not hang forever)
- ✅ Confirmed live: rejection, correction loop, retry counting, and graceful give-up all work

## Pilot-Requested Repeat ("say again" from the pilot)
Separate from the correction loop above. If the pilot says "say again"/"come again"/"repeat" (pattern-matched) while awaiting readback, the **original transmission replays verbatim** (not the "readback incorrect" wording), doesn't consume attempt budget, phase stays awaiting. ✅ Confirmed live.

Checked **before** the plausibility gate and validation, at all 3 levels, so it can never be misclassified as noise or a wrong attempt.

## Noise / Non-Attempt Filtering
A "plausibility gate" runs before counting anything as a real readback attempt: the transcription must contain at least one recognizable element relevant to a currently-pending phase, or it's **ignored entirely** (no attempt consumed, no "say again" triggered). Fixed a real bug where background noise/cross-talk was burning through the entire retry budget before a real attempt was possible.

## Phase-Change Resilience
A pending "awaiting readback" survives a phase change (even a flap through PARKED with no registered handler) — confirmed via a dedicated end-to-end regression test that drives a real `PhaseDetector` + `ATCEngine` through a debounce-confirmed flap.

## Interaction With Other Settings
- `ATC_TRIGGER_MODE` (automatic/ptt, Clearance-only) and `DISPATCH_TRIGGER_MODE` (automatic/frequency_required, all 7 non-ATIS phases) are **orthogonal** to `READBACK_VALIDATION_LEVEL` — all combinations tested and work.
- Readback-gating only begins **after a transmission actually plays** — a transmission withheld by the frequency gate does NOT start an awaiting-readback wait.
