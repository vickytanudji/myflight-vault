---
tags: [testing, setup]
---

# Test Setup & Methodology

## Standard Config
- **Airport:** YSSY (Sydney Kingsford Smith)
- **Callsign:** manual override **cleared** (empty) in `.env` — exercises the real SimConnect-derived callsign path (airline+flight number, falling back to tail number)
- **PTT key:** `grave` (backtick)
- **Aircraft:** varied across sessions (Cessna 172, Boeing 787/B78X) — deliberately, to catch per-aircraft telemetry gaps

## Required Startup Sequence (changed this session — read this)
1. Load MSFS to the **main menu** first if testing [[Flight-Load-Gate]] behavior — confirm no ATIS/ATC activity.
2. Load a flight **cold and dark** (engines off, not "ready to fly") — required for [[Phase-Detector]]'s CLEARANCE trigger to work correctly.
3. Confirm `CAMERA STATE` reaches `2` (cockpit) and the gate opens (~1.5s later).
4. Normal flow from there: Clearance → Ground → Tower Departure → Departure → etc.

## Key Settings to Toggle Per Test
| Setting | Values | Affects |
|---|---|---|
| `DISPATCH_TRIGGER_MODE` | `automatic` / `frequency_required` | Whether pilot must tune the right frequency — see [[Frequency-Tuned-Dispatch]] |
| `ATC_TRIGGER_MODE` | `automatic` / `ptt` | Clearance-only PTT-priority dispatch |
| `READBACK_VALIDATION_LEVEL` | `presence_only` / `keyword_match` / `full_icao_phraseology` | See [[Readback-Gating]] |
| `FLIGHT_LOAD_GATE_ENABLED` | `true` / `false` | Kill switch for [[Flight-Load-Gate]] |

## Manual Trigger Test Harness
```powershell
python -m tools.send_ws_request --type request_clearance
```
Also supports `--raw "..."` for malformed-input testing.

## Diagnostic Probe Scripts (all standalone, throwaway-style, all in `tools/`)
| Script | Purpose | Live-tested? |
|---|---|---|
| `probe_com1_frequency.py` | COM1/COM2 frequency read reliability | ✅ Yes — 2 aircraft |
| `probe_msfs2024_smoke_test.py` | Full SimVar smoke test against real MSFS 2024 | ✅ Yes |
| `probe_ai_traffic.py` | AI/FSLTL traffic enumeration | ❌ No — synthetic self-test only |
| `probe_chartfox_chart_shape.py` | ChartFox response shape inspection | ❌ No — needs credentials on PC |
| `diff_msfs_simvar_docs.py` | Reproducible 2020 vs 2024 SDK doc diff | N/A (doc-only, runs on Mac) |

## What "Passing" Looks Like in Logs
A clean session shows, roughly in order: SimConnect connect → camera-state gate open → ATIS → phase transitions (PARKED→CLEARANCE→GROUND→...) → transmission generation → (frequency check if enabled) → playback → readback wait → readback resolution. Deviations from this are either a known quirk (see [[08-Gotchas-And-Quirks]]) or worth investigating.
