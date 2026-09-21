---
tags: [overview]
---

# Project Overview

## What MyFlight Is
An AI-generated, phraseology-accurate ATC add-on for MSFS. It:
1. Polls live SimConnect flight data
2. Runs a phase detector to determine flight stage (parked, taxiing, departing, cruising, arriving, going around, etc.)
3. Generates real ICAO-phraseology ATC transmissions via a local LLM
4. Speaks them via TTS
5. Accepts PTT-triggered pilot voice input transcribed via STT

## Stack
- **Voice:** faster-whisper (STT, CPU/int8) + Kokoro ONNX (TTS, default voice `af_sarah`) + local LLM at `http://localhost:1234/v1` (`google/gemma-4-e4b`, LM Studio)
- **Flight data:** `python-SimConnect==0.4.26` against a running MSFS session
- **Chart data:** ChartFox API v2 (OAuth) for real SID/STAR/approach names
- **Real airport data:** OurAirports (runways, frequencies) — public domain, no API key

## Machines
- **Mac:** development, runs `cc-sonnet` (Claude Code), no live MSFS access
- **PC (Windows):** runs live MSFS, the *only* place live tests happen

## The 9 ATC Phases
ATIS, Clearance, Ground, Tower Departure, Departure, Enroute, Approach, Tower Arrival, Go-Around.

All 9 now use **deterministic ICAO templates** (see [[Readback-Gating]] and phraseology notes below) instead of pure LLM generation for the normal case — the LLM is only used for a narrow FREE_RESPONSE fallback per phase, and for the fully open-ended `handle_pilot_transmission` path.

## Phraseology Template Confidence
Stored in `docs/phraseology_reference.md`. Only **Clearance, Ground, Tower Departure** are CONFIRMED against real ICAO Doc 4444 text. The other 6 (ATIS, Departure, Enroute, Approach, Tower Arrival, Go-Around) are PROVISIONAL — real-world convention, not independently verified against Doc 4444 primary text.

> [!note] Deliberately deferred
> Full Doc 4444 / LiveATC validation of the 6 PROVISIONAL templates was explicitly deferred to **"polish later, after the app is functionally done."** Not a current priority.

## Test Aircraft / Callsign Convention
- Airport: **YSSY** (Sydney Kingsford Smith)
- Manual callsign override: cleared (testing the **real** SimConnect-derived callsign path, not `Global 123`)
- PTT key: `grave` (backtick)
- **Must start MSFS cold-and-dark** (not "ready to fly") — see [[Test-Setup-And-Methodology]]
