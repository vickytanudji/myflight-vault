---
tags: [bugs, history]
---

# Bug Log — This Session

## Connection handle leak on `AircraftRequests()`/`Request()` failure + related SimConnect robustness gaps
**Status:** ✅ Fixed, merged to `develop`. 17 new tests written first (15 failed on unfixed code, confirming they test something real). All 97 SimConnect tests pass. **MSFS 2020 live retest still outstanding.**
All four gaps closed:
1. **Adjacent leak (3a):** failures at all three exit points now route through one `_abandon_attempt()`, closing only the handle that specific attempt opened.
2. **Silent swallow (3b):** `stop_polling()` now logs a WARNING on `exit()` failure. The broad `except Exception` was deliberately kept (not narrowed) — `main.stop()` runs `chartfox.close()`/`ws_server.stop()` right after, and narrowing risks an unexpected exception skipping both.
3. **Unguarded verification read (3c):** now guarded, treated as an ordinary failed connection attempt; `reconnect_loop` survives a raising read.
4. **Event-loop stall (3d):** `reconnect_loop` now calls `await asyncio.to_thread(self.connect)`. Confirmed a concurrent coroutine keeps running during a stall.

**New follow-up items found, correctly left out of scope:**
- `_run_or_skip`'s skip logic is broader than ideal — skips on any `APIStatusError`/`ValueError`/timeout, not just "no model loaded," so a real 500 with a model loaded could silently skip instead of fail. Pre-existing.
- `stop_polling()`'s early-return: a connected-but-never-polled handle isn't closed on shutdown — a different leak scenario, not yet fixed.
- Shutdown delay: if `reconnect_loop` is cancelled mid-attempt, the worker thread still finishes (up to 10s) before `asyncio.run` returns — same worst case as the old blocking behavior, not worse, just not eliminated.

## Go-Around free-response signature mismatch
**Status:** ✅ Fixed, merged. **Root cause: the TEST was stale, not production.** `go_around.py`'s real call site (`_resolve_vectors_freq(flight_data, freq_data)`) was always correct — 12 existing offline tests already exercised it correctly. The failing test called the function with the pre-async-refactor signature (no args, no await), written correctly at the time, made stale 83 minutes later when the function became async. It has been silently skipped ever since (LM Studio unreachable on Mac) so its broken body never actually ran until LM Studio became reachable. **No production bug ever existed.** Reproduced exactly, then fixed with a shared helper + a new always-on test against a stub client (doesn't need a real LLM).

## `requires_real_llm` skip decorator doesn't handle "reachable, no model loaded"
**Status:** ✅ Fixed, merged. **The actual mechanism was different than first assumed:** `_run_or_skip` already correctly skipped on 400s for most tests. The 7 real failures were specifically the `handle_pilot_transmission` tests — that method **swallows every LLM exception internally and returns `""`**, so the 400 error never reaches the test's skip-detection at all; it just fails on `assert response` against an empty string.
**Fix:** a module-scoped pre-flight check sends one 1-token real request and skips only on a 400 whose body specifically says "no model loaded" — every other outcome (200, 5xx, timeout, connection error, a different 400) gives no verdict, so genuine failures still fail normally. Verified against a fake LM Studio server reproducing the exact original symptom (8 failed/8 skipped/2 passed → fixed to 3 passed/16 skipped for that subset).
**New item flagged, not fixed:** `handle_pilot_transmission` silently swallowing every LLM error and returning `""` is itself a production robustness gap — a pilot would get total silence with no "unable/standby" fallback on a real LLM failure. Worth a future look.

Chronological, most-recent-relevant first. Each entry: symptom → root cause → fix → live-confirmation status.

## `ATC RUNWAY AIRPORT NAME` returns display name, not ICAO code
**Symptom:** MSFS 2024 smoke test showed `'Kingsford Smith Intl'` instead of `YSSY`.
**Root cause:** field was never ICAO-shaped — neither SDK version's docs ever claimed otherwise. Wrong assumption originated from a mislabeled line in `docs/sdk/simconnect-variables.md`.
**Impact:** silently active this entire project's history, on both 2020 and 2024. A display name skipped fallbacks (worse than empty string) for Approach, Go-Around, ATIS, arrival-side frequency gate.
**Fix:** validate at source in `client.py` — only accept exactly-4-uppercase-letter values.
**Status:** ✅ Confirmed fixed live on MSFS 2020.
**See:** [[SimConnect-Client]]

## Connection handle/thread leak on failed verification read
**Symptom:** found by code-reading during 2024 smoke-test work; confirmed live when a zero-sim-running test threw `AttributeError: 'SimConnect' object has no attribute 'timerThread'` during cleanup.
**Root cause:** `connect()` dropped `self._sm` without calling `exit()` when the verification read returned `None`.
**Fix:** `_release_simconnect()` helper added, catches both `AttributeError` and `OSError`.
**Impact assessed:** ~12 leaks/min if hit repeatedly; likely does NOT affect the main-menu scenario (MSFS answers polls there, so `connect()` succeeds without retrying).
**Status:** ✅ Fixed, not live-tested (low urgency — narrow trigger window).
**Still open:** same leak class in a second branch; unguarded verification read; no timeout on the underlying wait at all (see [[SimConnect-Client]]).

## Go-Around vectoring frequency bug
**Symptom:** Go-Around fell back to a synthetic frequency at YSSY despite real Approach/Tower data existing.
**Root cause:** `_resolve_vectors_freq` read `flight_data.current_airport_icao` directly (known-unreliable field) instead of using `_resolve_arrival_icao` like the frequency gate and Approach's Tower handoff do.
**Fix:** one-line change to use `_resolve_arrival_icao`.
**Status:** ✅ Fixed, regression test written test-first (confirmed failing before fix).
**Note:** only reachable via Go-Around's free-response path (unresolvable callsign trigger) — if the fallback is seen again with a *resolvable* callsign, that's a different bug.

## SID name cleanup left multiple trailing parentheticals
**Symptom:** spoken clearance said "via the FISHA ONE (JET) departure" — should be "Fisha One."
**Root cause:** cleanup function only stripped the *last* trailing parenthetical, not all of them (some ChartFox names have two: nav-type + equipment-type tags).
**Fix:** loop-strip all trailing parentheticals.
**Status:** ✅ Confirmed fixed live.

## Readback Level 2 — digit/fuzzy matching too strict
**Symptom:** genuinely correct readbacks ("United 8-4-2... Fischer-1... Squawk 5-0-75") rejected as missing elements.
**Root cause:** naive exact/near-exact matching didn't normalize digit-word/numeral/hyphenated forms, and required exact SID-name spelling despite STT homophone noise.
**Fix:** digit normalization + fuzzy SID/destination matching, without weakening genuine-wrong-value rejection.
**Status:** ✅ Confirmed fixed live (exact failing transcripts now pass).

## Readback runway-span parsing gap
**Symptom:** "runway 3, 4, right" not recognized as matching expected "34R."
**Root cause:** matcher only had two candidate forms (full spoken phrase, or designator glued into one token) — missed the common STT shape of merged-numeral + separate-suffix-word.
**Fix:** dedicated inverse-of-`spoken_runway()` parser, ICAO-range-bounded, format-tolerant but not correctness-tolerant.
**Status:** ✅ Confirmed fixed live.

## Readback multi-pending misrouting
**Symptom:** with Clearance + Ground both pending, ambiguous/garbled input always routed to Clearance regardless of actual content.
**Root cause:** "closest partial match by missing-element count" heuristic ignored content/structure signals.
**Fix:** check content signals (taxi phraseology → Ground-family; SID/squawk shape → Clearance) before falling back to the count heuristic.
**Status:** ✅ Confirmed fixed live (reversed-order test: Ground-shaped readback correctly routed to Ground even with Clearance also pending).

## Readback attempt-counting consumed by noise
**Symptom:** irrelevant PTT input ("Yeah, when he did it, bruh!") counted as a full readback attempt, burning retry budget before a real attempt was possible.
**Fix:** plausibility gate — must contain ≥1 recognizable element relevant to a pending phase, or it's ignored entirely (no attempt consumed, no correction triggered).
**Status:** ✅ Confirmed fixed live (exact noise transcripts now ignored).

## Spurious CLEARANCE↔GROUND phase flapping
**Symptom:** stationary aircraft with engines running flapped between CLEARANCE and GROUND 5+ times/minute due to ground-speed sensor noise near the 1.0kt threshold.
**Fix:** redesigned boundary to use real external-power/APU/engine state instead of speed (see [[Phase-Detector]]).
**Status:** ✅ Confirmed fixed live — zero flaps over 90+ second stationary test.

## Secondary flap: CLEARANCE↔PARKED (power-state noise)
**Symptom:** after fixing the above, a *new* single-poll flap appeared on the PARKED boundary — `external_power_on` toggling true→false→true within ~5 seconds at session start.
**Investigation conclusion:** likely genuine cold-start telemetry transience (not sensor noise in the ongoing sense) — a debounce was added anyway (5 consecutive polls required) as defense-in-depth.
**Status:** ✅ Debounce added and confirmed live — no further flaps observed.

## Adversarial LLM bugs (`handle_pilot_transmission`)
**Bug 1 — value fabrication:** asked "what squawk am I assigned?" with none in context, LLM invented one AND appended an unrequested, out-of-scope taxi clearance.
**Bug 2 — out-of-authority compliance:** asked Clearance to "clear me to land," it complied (Clearance has no such authority).
**Fix:** strengthened `handle_pilot_transmission`'s system prompt with explicit declarative (not conditional) instructions — declarative framing proved more reliable for this model size.
**Status:** ✅ Confirmed fixed against the real local LLM, 18/18 adversarial tests passing twice in a row.
**Known, documented, unfixed gap:** post-hoc validators check *presence* of elements, not *correctness* — a transmission with a correct value plus a second, contradicting fabricated one would NOT be caught. Deliberately left as a known limitation, not fixed.
