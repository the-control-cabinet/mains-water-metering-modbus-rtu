# Changelog

How the design evolved. Dates are 2026.

## Hardware and comms
- **16-17 Nov:** SEN0257 pressure calibrated at the pin (ESP32 ADC ~15 mV low, gain ~0.2 %; divider 0.662; zero 0.480 V; net +0.025 bar at 3 bar); 10 µF at GPIO9 (noise 25-45 % lower); span confirmed without a reference gauge from the 5.87 m height to the cylinder inlets (overnight difference 0.55-0.58 bar = 0.576 bar), so no span gain (doc 04).
- **18-19 Nov:** cross-check package fixed and republished (no value instead of the text "unknown", reading date cut to the day, nightly snapshot skips a non-numeric night); TUF at 37-52 % of the smart meter 13-17 Nov; 17 Nov analysed: every draw counted but under-read, reverse total = at-rest jitter, the scale conclusion stands; bucket test next (doc 08 §8).
- **13 Nov:** cross-check and tariff packages added (account reference redacted); shower sub-metering fixes (flow keeps it running, humidity confirmation +6, status sensor).
- **12-16 Nov:** daily cross-check against the water company's smart meter (≈50 % low on 12 Nov); travel-time ratio 102.7 % at rest vs 108-109 % in cold draws; the meter's sound-speed estimate (1,367 m/s) below real cold water; plan: one-off spacing/pipe-settings check, then a ratio-based correction (new doc 08). Mains sag at shower flows ~0.5-0.6 bar (README corrected). Valve names written in full.
- **11 Nov (evening):** this meter's flow and pressure used to diagnose the cylinders: a 1-1.5 L draw at 18:31 explained a cylinder settling at ~2.5 bar (pressure reducing valves); flow added to the hot water card to mark draws; roadmap: pressure reducing valve comparison and overnight mains pressure.
- **11 Nov:** roadmap: mains water temperature NTC (cold-feed temperature, also feeding the heating repo's hot water estimate).
- **Sep:** TUF-2000M Modbus link proven on the utility-room ESP32-S3 (MAX485, 9600 8N1, slave 1). SEN0257 pressure sensor selected and plumbed using a BSP tee and a Dowty washer.
- **5 Oct:** pressure sensor moved to GPIO9 (ADC1). Formula corrected to the 16 bar full scale.
- **5-8 Oct:** ESPHome secrets migrated to `/config/esphome/secrets.yaml`.
- **21-22 Oct:** TS-2 transducers clamped on the 22 mm copper main (W method, 24.41 mm). Meter configured and settings saved with M26.

## Flow correction (all flashed or trialled)
1. Single multiplier 2.25 → 2.26, from totals only.
2. Line `1.82 × meter + 0.98`. Over-corrected slow flows (+21 %).
3. Zero reset (M43 / M42). Multiplier 2.17: slow draws 9 % low.
4. Quadratic ratio model, with caps above 8.5 / 12.4 / 20 L/min. Superseded when saturation was found.
5. 6-point lookup table.
6. **8-point lookup table with dead-band edge points (current).**

## Modbus
- Corrected the off-by-one addressing (manual number − 1) and `FP32_R` word order. Float total moved to address 114. Integer and fraction totals removed.
- Strength scale corrected from 0.0488 to 0.0244.
- Second, slow `modbus_controller` (60 s) added for diagnostics, replacing the retired `skip_updates`.
- `send_wait_time: 400ms`, `turnaround_time: 100ms` added to stop the 2 s bus stalls on ESPHome 2026.9.1.

## Draw tracker
- Per-draw raw-litre integration added. This enabled the clean Phase C calibration.
- Start threshold lowered from 0.8 to 0.25 L/min. End gap reduced from 30-60 s, to 3 s, to 2 s. Tick changed to 0.5 s.
- Live m:ss stopwatch added, plus the +0.001 L nudge for identical draws.
- Drift guard window shortened from 180 s to 120 s.

## Home Assistant
- Utility meters repointed to the corrected total, given `unique_id`s and recalibrated to 0.
- Cost sensors, bill-period counter, shower and washing-machine sub-metering, and the steady-low-flow alert added.
- Dashboard rebuilt as a 3-column Sections view with Gauge Card Pro (square-root green zone), a status and bill column, a draws chart and a pressure trend.
- Recorder `purge_keep_days: 10`.

## Published configuration
- **28 Oct:** ESPHome YAML brought in line with the live board: unused MQTT removed, INFO logging, debug sensors (heap, loop time).
