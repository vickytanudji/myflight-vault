---
tags: [gotchas, reference]
---

# Gotchas & Quirks — Read Before You Get Confused

## Testing Methodology Has Changed
- **Start MSFS cold-and-dark, not "ready to fly."** CLEARANCE now depends on real engine/power state (see [[Phase-Detector]]) — starting with engines already running skips CLEARANCE entirely.
- **Sitting at the main menu no longer produces fake activity** (see [[Flight-Load-Gate]]) — if you see nothing happening, check you've actually loaded into a flight, not just connected.
- If a test aircraft doesn't expose power telemetry, there's a ~20 second fallback before CLEARANCE engages anyway — don't mistake this delay/log line for an error.

## `current_airport_icao` Is Almost Always Empty Now
This is **correct, not broken** — see the display-name fix in [[SimConnect-Client]]. Don't write new code assuming it will populate. Use `resolve_departure_icao` (GPS-based) instead.

## Center/ARTCC Frequency Will Always Be Fake
Permanent, accepted limitation. See [[Center-Frequencies]]. Don't re-investigate this without new information.

## Camera-State Enum Values Differ Between Sim Versions
`9` = "Showcase" (2020) vs "Waiting" (2024). **Never hardcode a menu-value list** — the flight-load gate deliberately only checks the two values that mean the same thing in both sims: `{2, 3}`.

## SimConnect's Wrapper Silently Returns `None` for Missing Table Entries
This is not "the SimVar doesn't exist" — it means the wrapper's static table doesn't have it. Always check via a raw `SimConnect.Request` before concluding a field is unavailable. This pattern has bitten this project **at least 4 times** (runway airport name, external power, camera state, and would have for Facility Data/AI traffic too).

## Investigation Branches Are Disposable
Findings get carried forward via docs/conversation, but the git branches themselves are **deleted after review**, never merged. If a future brief needs a prior investigation's facts, **restate them inline** — don't reference the branch, it's probably gone.

## `docs/sdk/simconnect-variables.md` Was Stale, Not Tampered
Investigated and confirmed: the file's wrong unit-conversion claims (radians vs degrees, fps vs fpm) were **wrong from the original commit**, written before real testing found the correct behavior, and just never updated afterward. Not injected/tampered. Already fixed.

## Two Scaffold Doc Files Were Deleted
`docs/architecture.md` and `docs/atc-phraseology.md` were 0-byte placeholders since the initial commit, never populated, nothing referenced them. Deleted. Real phraseology docs live in `docs/phraseology_reference.md`.

## PTT-Release Latency Is a Known, Accepted, Deferred Issue
2–20+ seconds measured against the 3–5s target. **Explicitly accepted as a local-LLM-inference/model-size limitation**, deferred to a future cloud/faster-model fix. Not something to keep re-flagging.

## Manual Trigger Bypass Scope (memorize this)
The manual `request_clearance` trigger bypasses **phase detection** and the **frequency gate** — by design, that's the point of a manual override. It does **NOT** bypass the **flight-load gate** or the **one-shot guard**. If you're deciding whether a new manual-trigger-adjacent feature should bypass something, this is the precedent to follow.
