# Changelog

How the design evolved. Dates are 2026.

## Hardware and comms
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
