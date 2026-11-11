# 07 · Roadmap and Open Items

[← Back to README](../README.md)

## Validation

- [ ] **Missing-data test:** do a 10 L draw just after a 60 s stats poll. Compare the rise in `Meter Stats - Total (Positive)` with `Last Draw Raw Volume`; expect them within about 2 %.
- [ ] **Water-company dial check:** 142 m³ on 20 Oct 2026. Re-read after a few weeks and compare with `Mains Water Used`.
- [ ] **Yorkshire Water smart meter** (HACS `ha-yorkshire-water`, awaiting activation):
  - compare daily totals for the same date, allowing for the reporting lag
  - add a % difference card
  - scale the lookup table if the bias is consistent
- [ ] Scale-weighed high-flow container test.
- [ ] Watch M08, M90 and M91 during a full-open outside draw to diagnose saturation.
- [ ] Measure the toilet cistern volume (isolate, refill by jug) to check the 3.2 L flush reading.

## Monitoring and alerts

- [ ] **Sensors-slipped alert:** quality, travel-time ratio, strengths and error code, held for 10 min. Set thresholds from 1-2 weeks of baseline data.
- [ ] **Pipe temperature NTC:** 10 kΩ, B 3977, on GPIO4 or GPIO5 with 100 nF across it. The YAML is already in the file, commented out. Add a status row and a 24 h graph.
- [ ] Confirm the gauge flow bands (8 / 18 / 20.1 L/min are first guesses) after a few days of data.

## Pressure

- [ ] Explain the 3.06-3.6 bar oscillation at rest, which has a period of 1-2 minutes. Investigation steps:
  - shut the stopcock for 5 min and watch whether it continues
  - overlay the trace with hot-water cylinder heating
  - check the span against a mechanical gauge (new full-scale = 16 × gauge ÷ HA reading)
  - confirm a steady 4.5-5 V supply on the sensor's red wire
- [x] ~~Plumber: replace the expansion vessels~~: done 11 Nov (24 L, 3.5 bar; relief valves kept).
- [ ] **Cylinder PRVs settle at ~2.5 bar** against this meter's ~2.98 bar mains (both marked "pre-set 3.5 bar"); a full-flow kitchen draw dipped the left cylinder to 2.1 bar. Planned: a full-flow hot draw on each side for 30 s, comparing each cylinder's pressure drop against this meter's flow rate (bar per L/min), and a week of overnight mains pressure (does the mains ever exceed 3.5 bar?).

## Water temperature

- [ ] **Mains water temperature NTC** on the incoming main in the utility (strap-on, lagged, on the utility ESP32-S3's spare ADC1 pin). Seasonal cold-feed temperature (≈8-18 °C), captured during draws of 30-60 s (the pipe drifts to room temperature at rest), replacing the cold-feed helper (14 °C in November) in the [hot water available estimate](https://github.com/the-control-cabinet/heating-plant-monitoring-esphome/blob/main/docs/14-hot-water-stratification.md).

## Sub-metering

- [ ] Second shower: copy the master en-suite pattern with that room's motion and humidity sensors.
- [ ] Other showers.
- [ ] Dishwashers ×2. A detection method is needed (smart-plug power is the likely route).
- [ ] Shower start-rewind: count the first 30 s before "running" confirms.
- [ ] Oven and other appliances (deferred).

## Housekeeping

- [ ] Delete orphaned `unavailable` entities: the old integer, fraction and float totals, the old numeric duration, Draw Number and the stale meter-flow entity.
- [ ] Resolve the duplicate "(Mains Water)" device in HA.
- [ ] Optional: RS485 bias resistors to address the stray byte. Then file the ESPHome issue with logs.
- [ ] Optional: community write-up on TUF-2000M + ESPHome.

## Re-fit triggers

Re-run the calibration if any of these change:

- transducer position or couplant
- M45, M23, M24, M25 or M41
- the zero point (M42 / M43)
