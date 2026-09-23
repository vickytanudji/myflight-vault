---
tags: [todo, reminders]
---

# Standing Reminders — Check This Before Signing Off Each Session

## ⚠️ REGRESSION FOUND — Priority Fix Needed
- [ ] **SimConnect connection-robustness fix did NOT work as intended.** Live retest (2026-09-23) reproduced the exact `AttributeError: 'SimConnect' object has no attribute 'timerThread'` the original leak fix was supposed to eliminate — hit on every single retry attempt (16+ consecutive) with MSFS closed. The watchdog timeout itself works (fails fast, no hang), but the cleanup call afterward still throws, and the warning text itself says "the handle may not have been released." **This needs its own fix brief before trusting anything else in the connection-robustness area.** Also found in the same session: unexplained console spam of raw `SIM def(...)` lines during flight-load-gate polling — possibly related, possibly separate, needs investigation. See [[SimConnect-Client]] for full detail.

## Awaiting Live Retest Only (code merged, partially disproven above)
- [ ] **SimConnect connection-robustness fixes** (handle leaks incl. the never-polled-handle case, silent swallow, unguarded read, event-loop stall, watchdog timeout) — all merged to `develop`, fully pytest-verified (1218 passed, 16 pre-existing skips). **MSFS 2020 live retest still outstanding for all of it:** run the one-liner Python script, then `python -m core.main` with MSFS closed, then launch MSFS, then close/reopen it — specifically also test a shutdown right after a successful connect (before `start_polling()`), to exercise the newest fix. See [[SimConnect-Client]].
- [ ] **Enroute/Approach/Go-Around template content fix** — merged, 1219 tests passing. **Live retest owed:** confirm Enroute's restored cruise check-in, Departure→Center handoff happens exactly once (not duplicated across the two phases), Approach's combined descend+clearance+Tower-handoff sounds right, Go-Around's restored vectoring frequency is audible on the first attempt and uses a real value at YSSY (not the synthetic fallback). See [[Readback-Gating]] and [[Frequency-Tuned-Dispatch]] for how these phases interact with gating.

## Owed Live Tests
- [ ] **Frequency-tuning scenario (e)** — Center-sparse fallback (Enroute) + Departure/Go-Around real-frequency requirement. Needs a longer flight past Ground/Tower Departure. Scenarios (a)-(d) all confirmed. See [[Frequency-Tuning-Retest-Checklist]]. Note: Enroute's `CENTER` frequency-gate requirement is unchanged by the content fix — it gates on the *speaking controller's* frequency, not on what the transmission text says (see [[SimConnect-Client]] / frequency_gate.py's own documented design), and CENTER data is still sparse for YSSY, so this will likely still fall through to "satisfied" most of the time.
- [ ] **AI/FSLTL traffic probe** (`tools/probe_ai_traffic.py`) — needs the PC; some scenarios need real other traffic (VATSIM session or another human in multiplayer). See [[AI-FSLTL-Traffic]].
- [ ] What camera-state value a cold-and-dark **MSFS 2024** load reports for [[Flight-Load-Gate]] — never tested (2020 confirmed only). The smoke test never actually read `CAMERA STATE`, so this remains open even after the 2024 smoke test passed.

## Small/Cosmetic, No Urgency
- [ ] Dangling doc references in `frequency_gate.py`/probe scripts pointing at unmerged investigation-branch docs
- [ ] `CLAUDE.md`'s "Active work now (Month 4, Step 1)" paragraph is stale re: current-airport resolution
- [ ] Go-Around's re-armed second-attempt template (with its own re-sequencing-only, no-frequency content) is fully implemented and tested but still unreachable from live engine dispatch — `ATCEngine`'s one-shot guard on `GO_AROUND` is never re-armed. Separately tracked, not touched by the content fix.

## Deferred to "Polish Later" Phase (don't re-raise until app is functionally done)
- [ ] ICAO phraseology validation for the 6 PROVISIONAL templates against real Doc 4444/LiveATC — note Approach's descend-instruction placement is now a *deliberate scoping decision* (documented, PROVISIONAL confidence), not an oversight; worth a real citation check in this pass rather than before.
- [ ] Real Facility Data API investigation for anything (Center freq, AI traffic, procedures) — already ruled impractical multiple times; don't re-open without new information

## Vault / Workflow Housekeeping
- [ ] This vault lives at `github.com/vickytanudji/myflight-vault` (**public**, no auth needed to read) and is now edited **directly on Mac** via a filesystem connector — no more GitHub round-trip needed for Claude to update it. Push to GitHub / pull onto the PC via Obsidian Git remains the user's responsibility.
- [ ] Any GitHub PAT shared in chat for a push is single-use per session — Claude has no persistent credential storage. Regenerate/revoke tokens after use as a matter of course.
