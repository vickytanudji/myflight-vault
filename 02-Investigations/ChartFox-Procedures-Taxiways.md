---
tags: [investigation, closed]
status: CLOSED — procedure content NO-GO, taxiways DEFER
---

# Investigation: ChartFox Real Procedures + Taxiway Routing

Two independent questions, two independent answers.

## 1. Real SID/STAR Procedure Content — NO-GO

Today MyFlight only picks a real SID/STAR **name** via ChartFox — never the actual defined route (waypoints, altitude/speed restrictions).

**Findings:**
- **ChartFox (documented only, not live-tested — no credentials on Mac):** the saved API reference shows only chart metadata, no waypoints/restrictions/transitions.
- **FAA CIFP (confirmed, downloaded & decoded):** has real waypoint/restriction data, is public domain — but **US-only, and YSSY has zero records.**
- **MSFS SimConnect Facility Data:** the only global source matching what the pilot actually sees in-sim, but same wrapper gap as [[Center-Frequencies]] (not wrapped, needs custom ctypes work).
- **Why no-go regardless:** procedure content only pays off once the app can read the flight plan — **it can't**, so there's nothing to select against even with real data.

## 2. Real Taxiway Routing — DEFER

**Findings:**
- **OurAirports:** confirmed **no taxiway data at all.**
- **OpenStreetMap & X-Plane Scenery Gateway:** both give a real, connected, **named** taxi graph.
  - Naming completeness: OSM ~43% at YSSY, Gateway pack ~76%.
  - Full gate-to-runway route naming: conservatively 20–75%.
- **Licensing:** OSM is share-alike (needs legal read for a paid Cloud tier); Gateway's license reported as GPL-2.0 but unconfirmed from a primary source.
- **Layout mismatch risk:** both describe the *real* airport, which may differ from the sim's actual layout.
- **Facility Data (again):** the only source that would exactly match the sim, same wrapper-gap problem as above.

## Decision
- **Procedure content: no-go**, revisit only if/when real flight-plan reading is ever built.
- **Taxiway routing: defer** — genuinely feasible data exists, but naming coverage + licensing + sim-layout-mismatch risk make it "not yet," not "never." Revisit if either data source's coverage improves.
