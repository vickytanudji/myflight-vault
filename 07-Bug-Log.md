---
tags: [bugs, history]
---

# Bug Log — This Session

Chronological, most-recent-relevant first. Each entry: symptom → root cause → fix → live-confirmation status.

## APPROACH↔TOWER_ARRIVAL flapping (real, sustained oscillation on ordinary descents)
**Symptom:** repeated rapid cycling between APPROACH and TOWER_ARRIVAL during normal, unremarkable descents into YSSY — up to 4+ cycles per minute.
**Root cause confirmed:** Rule 7 (APPROACH: vs<=-500fpm) and Rule 8 (TOWER_ARRIVAL: vs<=-200fpm, alt<2500) share an overlapping vs band; since Rule 7's altitude condition is a superset of Rule 8's, TOWER_ARRIVAL can only ever be reached in the narrow (-500,-200] band. Real descent vertical speed genuinely oscillates across -500fpm from ordinary control/sensor noise — confirmed via two separate real flights with materially different noise widths (~-401 to -661 on one, ~-327 to -933 on another, nearly 300fpm wider).
**First fix attempt — insufficient:** a fixed one-sided hysteresis margin (200fpm, requiring vs<=-700 to revert to APPROACH) was tuned against only the first sample and failed live against the second, wider sample.
**Real fix:** combined a wider magnitude margin (600fpm, revert threshold -1100fpm) WITH a boundary-specific reversal debounce (4 consecutive polls) — reasoned explicitly that transient noise is a short-lived spike while a genuine reversal is sustained by definition, so combining magnitude and duration generalizes better than scaling magnitude alone (the required margin had already grown ~2.5x between just two samples with no evident ceiling).
**Status:** ✅ Confirmed fixed live across TWO independent full flights. Notably, one genuine sustained reversal (-1759fpm, a real level-off) was correctly allowed through as a real transition, not blocked — confirming the fix doesn't over-suppress legitimate reversals.

## Missed go-around via a brief on-ground blip
**Symptom:** a real go-around performed during a low pass was never detected as GO_AROUND at all — the aircraft's `on_ground` reading flipped briefly to `True` at very low altitude with high ground speed and still-negative vertical speed, and the phase detector read the subsequent climb as an ordinary takeoff (`TOWER_ARRIVAL → TOWER_DEPARTURE → DEPARTURE`) instead of a go-around.
**Root cause confirmed as a genuinely new case, not a broken existing gate:** the speed-divergence gate's 30kt threshold and the deliberate altitude-continuity skip-on-ground-flip (needed to avoid the historic frozen-baseline bug) both worked exactly as designed for their own purposes — neither was meant to catch this specific real-world scenario.
**Fix:** new `_go_around_eligible()` helper — `TOWER_DEPARTURE` is now also treated as go-around-eligible when the immediately-preceding confirmed phase was APPROACH/TOWER_ARRIVAL/GO_AROUND. Real takeoffs (always preceded by GROUND) and real landings (never leave `on_ground=True`) are structurally unaffected.
**Status:** ✅ Confirmed fixed live across TWO independent flights — both the easy case (a go-around that stays airborne the whole time, direct `TOWER_ARRIVAL → GO_AROUND`) and the hard blip case (`TOWER_ARRIVAL → TOWER_DEPARTURE → GO_AROUND`) confirmed working.

## `handle_pilot_transmission` silently swallowed LLM errors, returning `""`
**Status:** ✅ Fixed, merged. **Confirmed symptom first:** `_voice_loop` only synthesizes/plays `if response:` — an empty string is genuinely silent, not some other visible failure.
**Fix:** the except-block now returns `SAY_AGAIN_FALLBACK_TRANSMISSION = "Say again."` (new frozen phraseology constant, `docs/phraseology_reference.md` §11, PROVISIONAL confidence like the existing §10 correction template) instead of `""`. Deliberately generic/non-phase-specific — the failure can occur before any phase data resolves, so inventing phase-specific content risks the same fabrication-under-uncertainty problem the earlier adversarial-testing fix addressed. "Say again" chosen over "stand by" because it actually prompts a retry rather than leaving the pilot hanging with nothing to do. Error logging (`logger.exception`, `send_error`) unchanged.
**Tests:** 22 new tests — every exception type `LLMClient.complete` can raise, all 9 phases, logging still verified, success path unaffected, plus an end-to-end voice-loop test proving the fallback reaches TTS (and that a genuine `""` elsewhere still stays silent — pinning that gate, not changing it).

## `_run_or_skip`'s skip logic was broader than intended
**Status:** ✅ Fixed, merged. Old catch: `(APIConnectionError, APIStatusError, asyncio.TimeoutError, ValueError)` — all skipped, independent of the pre-flight check.
**New line drawn (empirically verified, not assumed):**
- **Still skips:** `APIConnectionError` excluding its `APITimeoutError` subclass (confirmed via actually probing a refused port and a DNS failure — both raise exactly this, never `APITimeoutError`); and `APIStatusError` matching the pre-flight check's own exact "no model loaded" signature (covers the model unloading between the pre-flight check and this specific call).
- **No longer skips (now fails):** `APITimeoutError` — AsyncOpenAI's default connect timeout is 5s, well under the 30s outer bound, so a timeout here means the connection succeeded and something hung. Genuinely ambiguous case, resolved by failing (the brief's explicit tie-breaker: a false failure you can investigate beats a false skip that hides a regression). Also no longer skips: other `APIStatusError`s (real 500s), plain `ValueError` (malformed/empty completion), the outer `asyncio.wait_for` timeout.
**Tests:** 9 new offline unit tests, confirmed red against the old broad implementation first, then green. Pre-flight check itself untouched, its own 15 tests unaffected.

## `stop_polling()` leaked a connected-but-never-polled handle
**Status:** ✅ Fixed, merged. **Root cause confirmed:** `if not self._running: ...; return` skipped the entire handle-cleanup block below it — so `connect()` succeeding without a subsequent `start_polling()` left the handle open forever.
**Fix:** thread-join logic stays gated on `self._running` (normal path unchanged), but handle cleanup now always runs afterward. Extracted into a new `_close_handle()` helper, reusing the `_abandon_attempt()` pattern — deliberately keeps `stop_polling`'s broader exception catch (not narrowed to `AttributeError`/`OSError`) since `main.stop()` still runs `chartfox.close()`/`ws_server.stop()` afterward either way.
**Tests:** confirmed red on unfixed code first, then green. Full existing SimConnect suite (102 tests) passes unchanged.
**Still outstanding:** live MSFS retest for this specific shutdown-right-after-connect sequence — hard to fully prove via mocks alone, folded into the next connection-robustness PC session.

**Combined verification for all three above:** `pytest tests/` → 1218 passed, 16 skipped (same pre-existing real-LLM skips, none new — baseline was 1184/16). 34 new tests, all green. `black --check .` clean repo-wide. No `%`-style loguru introduced.

---

## Connection handle leak on `AircraftRequests()`/`Request()` failure + related SimConnect robustness gaps
**Status:** ✅ Fixed, merged to `develop`. 17 new tests written first (15 failed on unfixed code, confirming they test something real). All 97 SimConnect tests pass. **MSFS 2020 live retest still outstanding** (now bundled with the item above's retest too).
All four gaps closed:
1. **Adjacent leak (3a):** failures at all three exit points now route through one `_abandon_attempt()`, closing only the handle that specific attempt opened.
2. **Silent swallow (3b):** `stop_polling()` now logs a WARNING on `exit()` failure. The broad `except Exception` was deliberately kept (not narrowed) — `main.stop()` runs `chartfox.close()`/`ws_server.stop()` right after, and narrowing risks an unexpected exception skipping both.
3. **Unguarded verification read (3c):** now guarded, treated as an ordinary failed connection attempt; `reconnect_loop` survives a raising read.
4. **Event-loop stall (3d):** `reconnect_loop` now calls `await asyncio.to_thread(self.connect)`. Confirmed a concurrent coroutine keeps running during a stall.

## Go-Around free-response signature mismatch
**Status:** ✅ Fixed, merged. **Root cause: the TEST was stale, not production.** `go_around.py`'s real call site (`_resolve_vectors_freq(flight_data, freq_data)`) was always correct — 12 existing offline tests already exercised it correctly. The failing test called the function with the pre-async-refactor signature (no args, no await), written correctly at the time, made stale 83 minutes later when the function became async. It has been silently skipped ever since (LM Studio unreachable on Mac) so its broken body never actually ran until LM Studio became reachable. **No production bug ever existed.** Reproduced exactly, then fixed with a shared helper + a new always-on test against a stub client (doesn't need a real LLM).

## `requires_real_llm` skip decorator doesn't handle "reachable, no model loaded"
**Status:** ✅ Fixed, merged. **The actual mechanism was different than first assumed:** `_run_or_skip` already correctly skipped on 400s for most tests. The 7 real failures were specifically the `handle_pilot_transmission` tests — that method **swallows every LLM exception internally and returns `""`**, so the 400 error never reaches the test's skip-detection at all; it just fails on `assert response` against an empty string.
**Fix:** a module-scoped pre-flight check sends one 1-token real request and skips only on a 400 whose body specifically says "no model loaded" — every other outcome (200, 5xx, timeout, connection error, a different 400) gives no verdict, so genuine failures still fail normally. Verified against a fake LM Studio server reproducing the exact original symptom (8 failed/8 skipped/2 passed → fixed to 3 passed/16 skipped for that subset).

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
**Status:** ✅ Fixed, superseded/folded into the broader connection-robustness fix above.

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
