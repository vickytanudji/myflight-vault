---
tags: [todo, reminders]
---

# Standing Reminders — Check This Before Signing Off Each Session

## Owed Live Tests
- [ ] **Full `core.main` connection-robustness retest** — beyond the one-liner script (confirmed 3/3). Launch → close MSFS → reopen, and specifically a shutdown right after a successful connect but before `start_polling()` runs — this exercises the never-polled-handle fix, not yet tested live.
- [ ] **Frequency-tuning scenario (e)** — Center-sparse fallback (Enroute) + Departure/Go-Around real-frequency requirement. Needs a longer flight past Ground/Tower Departure. Scenarios (a)-(d) all confirmed. See [[Frequency-Tuning-Retest-Checklist]]. Note: Enroute's `CENTER` frequency-gate requirement is unchanged by the content fix — it gates on the *speaking controller's* frequency, not on what the transmission text says (see [[SimConnect-Client]] / frequency_gate.py's own documented design), and CENTER data is still sparse for YSSY, so this will likely still fall through to "satisfied" most of the time.
- [ ] **AI/FSLTL traffic probe** (`tools/probe_ai_traffic.py`) — needs the PC; some scenarios need real other traffic (VATSIM session or another human in multiplayer). See [[AI-FSLTL-Traffic]].
- [ ] What camera-state value a cold-and-dark **MSFS 2024** load reports for [[Flight-Load-Gate]] — never tested (2020 confirmed only). The smoke test never actually read `CAMERA STATE`, so this remains open even after the 2024 smoke test passed.

## Small/Cosmetic, No Urgency
- [ ] Investigate the console spam of raw `SIM def(...)` lines observed during the connection-robustness testing (2026-09-23) — traced to the wheel's own internal `RequestList.py` retry-and-log behavior (`Constants.py`'s logger via stdlib logging, not this project's code), now suppressed via `logging.getLogger("SimConnect.Constants").setLevel(logging.CRITICAL)` in the same fix. Confirm this doesn't hide anything else useful from that logger.
- [ ] Dangling doc references in `frequency_gate.py`/probe scripts pointing at unmerged investigation-branch docs
- [ ] `CLAUDE.md`'s "Active work now (Month 4, Step 1)" paragraph is stale re: current-airport resolution
- [ ] Go-Around's re-armed second-attempt template (with its own re-sequencing-only, no-frequency content) is fully implemented and tested but still unreachable from live engine dispatch — `ATCEngine`'s one-shot guard on `GO_AROUND` is never re-armed. Separately tracked, not touched by the content fix.

## Deferred to "Polish Later" Phase (don't re-raise until app is functionally done)
- [ ] ICAO phraseology validation for the 6 PROVISIONAL templates against real Doc 4444/LiveATC — note Approach's descend-instruction placement is now a *deliberate scoping decision* (documented, PROVISIONAL confidence), not an oversight; worth a real citation check in this pass rather than before.
- [ ] Real Facility Data API investigation for anything (Center freq, AI traffic, procedures) — already ruled impractical multiple times; don't re-open without new information

## Vault / Workflow Housekeeping
- [ ] This vault lives at `github.com/vickytanudji/myflight-vault` (**public**, no auth needed to read), synced between Mac and PC via Obsidian Git. Claude can edit it directly from either machine via a filesystem connector, depending on which is active in a given session.
- [ ] **⚠️ Real risk, already occurred once (2026-09-23):** editing the vault from both machines without pushing/pulling in between causes genuine git merge conflicts (nested `<<<<<<< HEAD` markers ended up in the actual file text, requiring manual cleanup). Push after any edit session before switching machines, or before starting a new Claude session on the other machine.
- [ ] Any GitHub PAT shared in chat for a push is single-use per session — Claude has no persistent credential storage. Regenerate/revoke tokens after use as a matter of course.
