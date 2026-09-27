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
Readback must contain key expected elements (callsign, runway, altitude, squawk, SID name, destination, frequency as applicable) — no strict phraseology required.

**Real bugs found & fixed via live testing (chronological):**
1. **Digit/fuzzy matching (squawk/SID)** — STT renders numbers many ways ("5-0-75" / "5075"); SID names get mis-transcribed ("Fisha" → "Fischer"). Fixed with digit normalization + fuzzy SID/destination matching.
2. **Runway-span parsing** — "3, 4, right" / "three four right" wasn't recognized as matching "34R". Fixed with a dedicated inverse-of-`spoken_runway()` parser, bounded to real ICAO ranges (01–36), never accepting an arbitrary parsed runway — only one matching the *specific expected* value (format tolerance, not correctness tolerance).
3. **Multi-pending routing** — with 2 readbacks pending simultaneously (e.g. Clearance + Ground), an incoming response was routed to the wrong one using a naive "fewest missing elements" heuristic. Fixed with content-signal-based routing (taxi phraseology → Ground-family; SID/squawk-shaped content → Clearance) before falling back to the missing-count heuristic.
4. **Frequency format matching (2026-09-27) — ✅ fixed, live-confirmed.** The frequency element only accepted the fully spelled-out TTS form ("one two seven decimal five") or the exact literal "127.500" — a genuinely correct real readback ("...contact center 127, decimal 5.") was rejected. Fixed using the same span-finder approach as runway: digits are read back through the shared tokenize/numeral-merge path and compared to the expected value. Now accepts fully separated words, "127, decimal 5", "127.5", "127.50", "point", and "decimal" dropped entirely. Wrong frequencies (126.5, 127.55, 137.5, or a wrong-precision match like 118.25 vs 118.025) are still correctly rejected. Covers Tower Departure, Departure, Approach's Tower handoff, and Go-Around's vectoring frequency — the four phases with a frequency element.
5. **Misheard "decimal" (2026-09-27) — ✅ fixed, live-confirmed.** STT mis-transcribed "decimal" as "disable" or "deserts" in real sessions. Fixed: any single non-digit word sitting exactly between the expected whole-number digits and the expected fraction digits is accepted as the decimal separator — EXCEPT words that start another instruction (runway, heading, squawk, contact, etc.), which prevents a false positive like "heading 127, runway 5" being misread as a frequency. Both digit groups must already exactly match the assigned frequency, so the separator word can never turn a wrong frequency into a right one — it only decides whether two already-correct numbers belong together. Deliberately still rejects "127 deserts" alone (no fraction digit present at all — accepting it would mean any frequency from 127.1–127.9 passes, which loosens *correctness*, not just format).
6. **Callsign flight-number format matching (2026-09-27) — ✅ fixed, live-confirmed.** Root cause was NOT the callsign matcher itself (which already accepted "United 842") — it was the **shared tokenizer**, which stripped every comma and merged adjacent numeral runs, turning `"842, 10,000"` into a single corrupted token `84210000`, breaking callsign AND altitude parsing simultaneously (explains why the original bug report showed both missing). Fixed: numbers either side of a comma now only merge when both sides are single digits (so "8, 4, 2" and "1, 2, 7" still merge; "842, 10,000" does not) — thousands-comma stripping (`10,000`) is unaffected. Also added: the callsign's flight-number portion now reads back through the same digit-word tables as frequency, so "eight forty-two", "eight four two", "842", "8-4-2", "8, 4, 2", "eight 42" all match, for both the SimConnect-derived callsign and `DEFAULT_CALLSIGN_OVERRIDE`. **Deliberately NOT loosened:** the airline-name portion stays an exact match — "We added 842" (STT garbling "United") is still correctly rejected, since "added" isn't meaningfully closer to "united" (similarity 0.364) than an unrelated real airline name would be (e.g. "unity" scores 0.727) — no similarity threshold could accept the garble without also risking accepting a genuinely different airline.
   - **Known trade-off from the tokenizer fix:** a squawk or altitude read back as comma-separated *multi-digit* chunks (e.g. "squawk 45, 21") no longer merges into one number. Not observed in any real log yet — flagged for awareness, not a confirmed problem.

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

## Stale Readback Cleanup — ✅ fixed, live-confirmed (2026-09-27)
The engine's existing cleanup logic (`"readback interaction ended — superseded..."`) only fired on a phase **re-arm** (e.g. a go-around). A normal, sequential one-way phase advance (e.g. DEPARTURE → APPROACH with DEPARTURE's readback never satisfied) did NOT clear the stale entry — it just sat there indefinitely, only papered over by the multi-pending router correctly guessing the right phase anyway.

**Fix:** when a transmission plays, any pending readback **two or more phases earlier** is now abandoned via the same log line/helper. **Adjacent phases stay simultaneously pending** (e.g. Clearance + Ground, a real and legitimate case) — only a phase that's been superseded by something *further along* gets cleared. Chosen specifically because a stricter "clear anything superseded" rule would have incorrectly cleared Clearance the moment Ground plays, breaking that legitimate adjacent-pending case.
- DEPARTURE → APPROACH (the original live bug): DEPARTURE now clears. ✅
- DEPARTURE → ENROUTE: does **not** clear DEPARTURE (only 1 phase later) — it clears once APPROACH plays instead. A known, accepted asymmetry, not a bug — changing it would break the Clearance+Ground case.
- Go-around re-arm path is completely unchanged — this fix is additive, not a replacement.

## Interaction With Other Settings
- `ATC_TRIGGER_MODE` (automatic/ptt, Clearance-only) and `DISPATCH_TRIGGER_MODE` (automatic/frequency_required, all 7 non-ATIS phases) are **orthogonal** to `READBACK_VALIDATION_LEVEL` — all combinations tested and work.
- Readback-gating only begins **after a transmission actually plays** — a transmission withheld by the frequency gate does NOT start an awaiting-readback wait.
