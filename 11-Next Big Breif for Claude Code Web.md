TASK: Systematically audit every phase-detection boundary in
core/atc/phase_detector.py for the SAME class of bug that has now been
found and fixed THREE separate times this project, reactively, one at a
time, via live MSFS testing: a phase boundary defined by a raw threshold
on noisy real telemetry, which flaps back and forth when real sensor
noise straddles that exact value.

CONTEXT — the three prior occurrences, for pattern-matching:
1. CLEARANCE<->GROUND: originally a raw ground-speed threshold (1.0kt) —
   flapped because stationary-aircraft ground speed noise straddled it.
   Fixed by redesigning the boundary around power/engine state instead
   of speed.
2. CLEARANCE<->PARKED: a secondary flap on external_power_on/apu_pct_rpm
   toggling at session start — fixed with POWER_STATE_DEBOUNCE_POLLS (a
   5-consecutive-poll debounce).
3. APPROACH<->TOWER_ARRIVAL: Rule 7 (vs<=-500fpm) and Rule 8 (vs<=-200fpm,
   alt<2500) share an overlapping vs band, and real descent vs noise
   genuinely oscillates across -500fpm on ordinary approaches (confirmed
   via two separate real flights, noise bands of roughly -401 to -661 on
   one approach and -327 to -933 on another — nearly 300fpm wider on the
   second). Fixed with a combination: a widened one-sided hysteresis
   margin (VS_BOUNDARY_HYSTERESIS_FPM) AND a boundary-specific reversal
   debounce (VS_BOUNDARY_REVERSAL_DEBOUNCE_POLLS = 4) — magnitude alone
   was proven insufficient across two real samples, since the required
   margin grew ~2.5x between them with no evidence of a ceiling; a
   transient spike is short-lived, a genuine reversal is sustained, so
   combining a magnitude check with a duration check generalizes better
   than scaling magnitude alone.

Given this pattern has now recurred three times with the same
underlying character (fixed thresholds vs. real noisy telemetry), it is
worth proactively auditing every OTHER boundary in this file rather than
waiting for a fourth live discovery.

REQUIREMENT:

1. INVENTORY every phase-transition rule in _detect_phase (the full
   9-rule table, or however many currently exist — read the actual
   current file, do not assume the rule count/order from memory) and
   every OTHER threshold-based transition-adjacent mechanism in this
   file (e.g. the exact-zero-quorum gate, the on-ground/airborne speed-
   divergence gates, the altitude-continuity thresholds, the go-around
   climb-vs threshold, ANY constant compared against a single live
   telemetry value to decide or gate a phase transition).

2. For EACH rule/threshold identified in step 1, classify it into ONE of:
   a. ALREADY PROTECTED — already has a debounce, hysteresis, or
      streak-based confirmation mechanism (like the three fixes above)
      that would plausibly prevent flapping from realistic sensor
      noise. State which mechanism protects it.
   b. VULNERABLE, SAME PATTERN — a single raw threshold on a telemetry
      value that plausibly has real-world noise characteristics similar
      to ground speed, power-state booleans, or vertical speed (i.e.
      values that hover near a boundary during normal, unremarkable
      flight, not just extreme edge cases). Flag explicitly WHY you
      believe it's vulnerable (what real flight scenario would produce
      noise near this exact threshold, analogous to the three confirmed
      cases).
   c. LOW RISK — a threshold that is either far from any value real
      telemetry would realistically approach during normal operation
      (e.g. a threshold only relevant to an extreme/rare state), or
      inherently defended by the SHAPE of its own logic (e.g. requires
      a large, unambiguous state change rather than crossing a fine
      boundary). Justify why you believe this classification is safe --
      do not default to "low risk" without reasoning, given three
      supposedly-safe boundaries have already turned out otherwise.
   Produce this classification as a clear, reviewable table/document
   BEFORE writing any fix code -- do not skip straight to
   implementation. Do not assume real-world telemetry noise
   characteristics you have not seen -- reason from the ACTUAL noise
   patterns already observed live in this project (documented in
   CONTEXT above and in this project's own test fixtures/regression
   tests for the three known cases) rather than inventing new noise
   assumptions.

3. For every rule classified as (b) VULNERABLE in step 2, design and
   implement a fix using the SAME toolkit already established and
   proven in this file (debounce polls, hysteresis margins, streak-based
   confirmation with a current-poll-signature hold, or a combination) --
   do not invent a fundamentally new mechanism unless you can clearly
   justify why none of the existing patterns fit. Follow the same
   discipline already demonstrated on this file: any fix must not
   regress a GENUINE, real transition's responsiveness unreasonably (a
   real go-around, a real landing, a real takeoff must still be
   detected in a reasonable number of polls) -- for each fix, explicitly
   reason about this tradeoff, the same way the three prior fixes did.

4. Do NOT modify any rule/threshold classified as (a) ALREADY PROTECTED
   or (c) LOW RISK -- only implement fixes for (b) VULNERABLE items.

5. Do NOT change the fundamental SEMANTIC MEANING of any rule (what
   real-world flight state each rule is trying to detect) -- only the
   CONFIRMATION MECHANISM (how quickly/robustly a transition is
   accepted).

TESTS:
- For every (b) VULNERABLE item you fix: a regression test using
  REALISTIC synthetic noise (modeled on the actual character of the
  three already-observed real noise patterns -- oscillation around a
  fixed value, not random jitter) proving the new fix prevents flapping
  under that noise pattern, AND a test proving a genuine, sustained
  transition still confirms within a reasonable poll count.
- Run the ENTIRE existing phase_detector test suite (every existing
  test class: TestPhaseDetection, TestPhaseTransition, TestPhaseDebounce,
  TestPlausibilityGate, TestExactZeroQuorumGate,
  TestPlausibilityRejectionLogDedup, TestGoAroundDetection,
  TestAltitudeBaselineRecovery, TestApproachTowerArrivalHysteresis, and
  any others present) -- ZERO regressions permitted, given this file's
  extensive and hard-won history.
- Confirm black --check . is clean and no %-style loguru calls are
  introduced.

OUT OF SCOPE: Any change to engine.py, frequency_gate.py, readback.py,
flight_load_gate.py, or any phase module outside phase_detector.py
itself. Any change to a rule's semantic meaning. Any change to an item
classified as (a) or (c).

DEFINITION OF DONE:
- A complete, reviewable classification of every threshold-based
  mechanism in phase_detector.py into (a)/(b)/(c), with reasoning for
  each -- delivered as part of your summary, not just implicit in the
  diff
- Every (b)-classified item fixed using this file's established
  toolkit, with reasoning for why that specific mechanism (debounce,
  hysteresis, or a combination) fits that specific boundary
- Zero regression to any existing test, including the three already-
  fixed boundaries from this session's history
- pytest and black both clean
- Your summary explicitly lists what STILL requires live MSFS
  confirmation (this is expected -- design/implementation can happen in
  the cloud, but per this project's own hard rule, NOTHING touching
  phase_detector.py is considered actually fixed until confirmed live
  on the PC by the user; do not claim otherwise)