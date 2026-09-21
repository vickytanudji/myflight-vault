---
tags: [investigation, closed, no-go]
status: CLOSED — NO-GO, permanent
---

# Investigation: Real Center/ARTCC Frequencies

## Question
Can real Center/ARTCC frequency data be sourced (via MSFS SimConnect Facility Data API, or an alternative) to close the gap where Enroute/Departure/Go-Around fall back to synthetic Center frequencies?

## Findings

### Q1 — Does the data exist in MSFS?
Only in pieces that don't close the gap:
- **Center airspace polygons:** `BoundaryType.Center` exists but only in the in-sim JS Avionics Framework — **not** exposed via SimConnect Facility Data in either 2020 or 2024 docs. The boundary record has **no frequency field** anyway.
- **Center-typed frequencies:** the `AIRPORT → FREQUENCY` record has a CENTER type, but it's **attached to an airport** — same shape as OurAirports' CNTR rows, so it can't tell you which Center applies en route (Center coverage is airspace-based, not airport-based).
- Density likely sparse (inferred from an FAA doc: NASR omits Center-to-airport links "across almost all ARTCC facilities").

### Q2 — Does `python-SimConnect` expose Facility Data at all?
**No.** `SimConnect==0.4.26` wraps only the older Facilities List API — does not wrap `AddToFacilityDefinition`/`RequestFacilityData`. The bundled DLL doesn't export either function. A raw ctypes route is theoretically possible but needs a different DLL, hand-written prototypes, and a dispatcher override.

### Q3 — Effort assessment
Facility Data route: **much larger and riskier** than any prior data-source work in this project (e.g. OurAirports integration). And it **wouldn't even fix Enroute** — see the Go-Around correction below.

### Q4 — Alternative sources
A bundled regional dataset (FAA NASR, US-only, public domain) is comparable effort to build, but:
- Doesn't cover YSSY (the test airport) or anywhere non-US
- VATSpy's FIR boundaries are CC-BY-SA-4.0 — needs legal review before bundling into a paid product

## Important Correction Found During Investigation
**Go-Around's frequency gate is NOT Center-related** — it's `(TOWER, APPROACH)`. The `DEFAULT_CENTER_FREQ_MHZ` name seen in early logs was just a fallback constant's *name*, unrelated to this investigation. (This later turned out to be a **real, separate bug** — see [[07-Bug-Log]] "Go-Around vectoring frequency bug".)

## Decision
**No-go, permanent.** Center-gating already degrades gracefully (auto-satisfied, plays normally) — not worth the complexity relative to benefit. Enroute is the only phase that genuinely needs this and it will simply always use its documented fallback.
