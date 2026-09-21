---
tags: [investigation, confirmed, pursue]
status: CONFIRMED WORKING — dual-support formalized
---

# Investigation: MSFS 2024 Support

## Recommendation
**Support both sims on one code path.** No migration, no indefinite deferral. Estimated 3–5 sessions of work given the first smoke test passed.

## Documentation Findings (pre-live-test)
- **All 25 SimVars `client.py` reads exist in the 2024 docs with identical units.** Only cosmetic doc differences (longer `ATC MODEL` string limit, clearer `EXTERNAL POWER ON` indexing).
- Microsoft's own 2024 SDK docs state a 2020-SDK-built client "will work as before." Community reports agree, but this exact pinned wheel had never been tested until this session.
- `python-SimConnect` has had **no commits since June 2022** — no 2024-specific version exists, none needed. Its table has no gaps for anything MyFlight reads (it's missing 295 of 296 SimVars **new** to 2024, all irrelevant to this project).
- Connection failures degrade to a generic "Did not find Flight Simulator running" message either way — not sim-version-specific, and not very diagnostic (see the watchdog gap in [[SimConnect-Client]]).
- **A 2024 upside:** `ATC DESIGNATED RUNWAY TAKEOFF/LANDING` closes a documented gap.
- Licensing question (can the bundled `SimConnect.dll` be redistributed?) remains **unanswered** — applies regardless of which sim version, standing open item.

## Live Smoke Test Results (`tools/probe_msfs2024_smoke_test.py`, real MSFS 2024, Steam, Windows 11)

**Run B (menu, `--wait`):** 30/30 PASS. Connected in 0.08s.

**Run C (loaded flight, YSSY, C172):** 28/30 PASS, 2 SUSPICIOUS:
1. `ATC_AIRLINE` empty — expected, already-documented "some aircraft don't set this" behavior, not new.
2. **`ATC RUNWAY AIRPORT NAME` returned `'Kingsford Smith Intl'`** — this was the discovery that led to the major display-name bug fix (see [[SimConnect-Client]] and [[07-Bug-Log]]). **Confirmed on MSFS 2020 too** — not 2024-specific, was present this entire project's history.

**Separately confirmed (zero-sim-running test):** SimConnect can accept a handle open even with literally no sim process running, then fail on first data read — a different, worse failure shape than expected. Also surfaced a cleanup-path `AttributeError` (`'SimConnect' object has no attribute 'timerThread'`) — related to the handle-leak fix, see [[SimConnect-Client]].

## Two Expected-Variance Items (documented as awareness, not bugs)
1. **2024 AI-traffic state behaves differently from 2020** — relevant to [[AI-FSLTL-Traffic]] work later.
2. **Some default 2024 airliners reportedly have external-power/APU telemetry issues** — same class of per-aircraft fragility already handled by the existing `POWER_TELEMETRY_FALLBACK_POLLS` fallback. No new code needed.

## Decision
**Confirmed, dual-support formalized.** No sim-version detection/branching code added anywhere — genuinely not needed, since the SimConnect surface is identical for everything this project reads. Docs updated (`CLAUDE.md` / `docs/sdk/simconnect-variables.md` / `AGENTS.md`) to state both versions are supported, including a real, documented third variance (`CAMERA STATE` enum numbering differs — see [[SimConnect-Client]]).

## ✅ Watchdog Timeout — Done
Connection watchdog (10s) added to production `client.py`, committed on `feat/formalize-dual-sim-support`. Mutation-tested. **MSFS 2020 live retest still outstanding** (script provided, not yet run).

## Still Owed
- MSFS 2020 live retest of the watchdog fix (one-liner Python script provided in the brief output)
- MSFS 2024 cold-and-dark camera-state value for [[Flight-Load-Gate]] — never tested (the smoke test never actually read `CAMERA STATE`)
- Two related connection-robustness gaps found during this same work, spun into a separate not-yet-run brief — see [[07-Bug-Log]]
