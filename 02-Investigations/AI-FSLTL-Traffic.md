---
tags: [investigation, pursue, live-test-pending]
status: PURSUE — gated on a live PC test (not yet run)
---

# Investigation: AI Traffic & FSLTL Awareness

## Correction Up Front
**FSLTL is an independent project, not FlyByWire's** — FBW is credited only for the installer it's distributed through.

## Key Finding: One Problem, Not Two
Native MSFS AI traffic and FSLTL-injected traffic can be read **the same way** — FSLTL's own docs say it injects traffic through SimConnect, and once injected "the sim is in control of the movement of AI aircraft." A single call, `SimConnect_RequestDataOnSimObjectType` (radius up to 200km, type AIRCRAFT), should return both. **Little Navmap does exactly this on MSFS today** (source code reviewed as precedent).

## Data Available Per Contact
Identity, position, speed, heading, vertical speed, ground flag, plus AI-specific `AI TRAFFIC STATE` and `AI TRAFFIC ASSIGNED RUNWAY` — the latter two could answer "who is landing on my runway" with no geometry needed, **if they actually populate reliably** (unconfirmed).

## FSLTL Specifically
Has **no data API of its own** — its `localhost:42888` REST API only *controls* the injector (settings, reset, cull, kill), doesn't expose traffic data. Nothing needs FSLTL installed; with no traffic you just get an empty list — **safe to build against even if the user doesn't have it.**

## Wrapper Gap
**Medium** (smaller than Facility Data). `python-SimConnect==0.4.26`'s high-level API is hard-wired to the user's own aircraft and keeps only one value per reply — won't work for enumerating others directly. But the **bundled DLL already exports what's needed** (unlike Facility Data) — no DLL swap required, just lower-level wheel access.

## Reliability Profile — Different, Not Lower
Per-aircraft-package unreliability (external power, APU) mostly doesn't apply here. New risks instead:
- `SIM_ON_GROUND` reportedly unreliable **for AI aircraft specifically** (per Little Navmap's own code)
- AI-specific variables empty without a flight plan
- AI traffic itself behaves oddly (wrong runways, stuck aircraft)
- **Recommendation: treat contacts as advisory hints, not hard gates.**

## Not Yet Confirmed Live
- Built-in MSFS multiplayer: **not readable** per Asobo forum statements (2026-07-30, "No ETA") — but those threads are about MSFS 2024; 2020 evidence is older, needs its own live check.
- VATSIM/IVAO pilots: probably readable (vPilot injects via SimConnect) — inference only, needs a real network session to confirm.

## Probe Script
`tools/probe_ai_traffic.py` (~1000 lines) — self-test passes 10/10 on synthetic data (pure logic only). **Never run against a live sim.** This is the gate before any V1 implementation work.

> [!todo] Standing reminder
> Run `tools/probe_ai_traffic.py` on the PC. Some checklist scenarios need real other traffic present (a VATSIM session, or another human in multiplayer) — see doc §9 for the full 21-item checklist.

## Decision
**Pursue**, but gated entirely on that live test. Minimal V1 sketch (from the doc, not yet built): a passive "nearby traffic" snapshot, no ATC behavior change yet.
