# 06 · Issues Dealt With and Lessons Learned

[← Back to README](../README.md)

Each issue is written up as **symptom → cause → fix → evidence**. They are grouped by layer.

---

## A. Measurement (the meter)

### 1. The meter reads less than half the real flow
- **Symptom:** a 16 L bucket registered 6.5-7.8 L. The shortfall was worse at low flow (34-41 %) than at high flow (about 48-52 %).
- **Cause (best-supported):** the TS-2 / TUF-2000M acoustic model doesn't fit 22 mm thin-wall copper. TS-2 is rated DN25+, and in a wall thinner than an acoustic wavelength the sound partly travels as a guided wave. Settings, zero, units and geometry were each tested and ruled out as the main cause ([docs/04](04-calibration-and-testing.md#why-it-under-reads-theories-tested)).
- **Fix:** M45 at its 1.5 maximum, then an **8-point piecewise-linear lookup table** in the ESP32, fitted from bucket tests.
- **Evidence:** 10 L verification draws at 11-15 L/min averaged 9.96 L.

### 2. The meter saturates at about 9.5 L/min displayed
- **Symptom:** a full-open outside tap gave *lower* ratios than a ⅔-open one. The meter's own flow sat at 9.3-9.6 L/min whether the true flow was 18.5 or 21 L/min.
- **Cause:** not yet diagnosed. Candidates are a timing-range limit on small pipe, air at speed, or signal lock.
- **Fix:** the table clamps at 20.08 L/min true, and the diagnostic **"Core - Flow At Meter Limit"** turns on at M ≥ 9.28 so saturated periods are visible.
- **Evidence:** two independent runs at 20.7 and 21.05 L/min.

### 3. Zero drift and phantom flow after a draw
- **Symptom:** after a draw, a small steady flow persisted. Raising the cut-off hid it but also hid slow draws. On one occasion the meter showed 0.45 L/min with nothing open, which cleared when another tap was used.
- **Cause:** a zero set while water was moving, plus baseline shift (couplant, mechanical, or air in the beam).
- **Fix:**
  - Reset the zero properly: **M43**, then **M42** with the water truly off, then **M26** to save. Confirm **M44 = 0**.
  - Lower the cut-off **M41** to 0.02 m/s and damping **M40** to 2 s.
  - Firmware **dead band** of 0.15 L/min.
  - **Drift guard:** below 0.9 L/min for more than 120 s is forced to 0.
  - **Steady-low-flow alert** in HA after 10 minutes.
- **Lesson:** in a firmware threshold, make the drift visible (a diagnostic binary sensor) rather than silently hiding it.

### 4. Transducer-type option made things worse
- **Symptom:** M23 = "Clamp-on TS-2" (option 19, listed as *reserved* in the manual) asked for 17.69 mm spacing. Left at 24.4 mm it read 37.4 % with M91 at about 110 %.
- **Fix:** returned to **Standard S** (48.6 %).
- **Lesson:** change one variable at a time, and record M90/M91 alongside every result.

---

## B. Communications (Modbus / RS485)

### 5. Every value read one register late
- **Symptom:** flow looked fine. Totals behaved strangely (see issue 6).
- **Cause:** the manual's register *numbers* (starting at 0001) were used as protocol *addresses* (starting at 0). The flow float was assembled from REG0002 plus REG0003 (which is zero), so it lost its low 16 bits. That cost only 0.4-0.8 %, which is why it looked right.
- **Fix:** **address = manual number − 1**, with `FP32_R` (low word first).
- **Lesson:** a value that is "nearly right" is a red flag for word-order and offset errors. Prove the map against a known value: the float total at address 114 matched the meter's M02 exactly.

### 6. Totaliser "jumped" by thousands of litres
- **Symptom:** the HA total went 37,739 → 38,326 → 31,326 → 42,650 → 4,045. Every value was below 65,536.
- **Cause:** the off-by-one read combined the integer total's high word with the *low word of the fraction float*, which is effectively a random 16-bit number. The float total also moved in exact 1/512 m³ steps because its low half was missing.
- **Fix:** read the positive total float at address 114 (`FP32_R`). The integer and fraction registers were dropped.
- **Lesson:** an elaborate theory about battery-backed RAM and offline compensation was proposed for these jumps and later **retracted**. Check the addressing before inventing physics.

### 7. Signal strength exactly double
- **Cause:** the manual's 0-2047 scale does not match this firmware. A raw count of 3,141 corresponds to a display of 76.6.
- **Fix:** multiply by **0.0244**. The first value tried, 0.0488, was exactly 2× too high.

### 8. ESPHome 2026.9: `skip_updates` no longer works
- **Symptom:** diagnostics polled every second regardless of setting.
- **Cause:** `skip_updates` was retired in the 2026.9 Modbus refactor.
- **Fix:** a **second `modbus_controller` at the same slave address** with `update_interval: 60s` for all "Meter Stats". The fast controller keeps flow at 1 s.

### 9. ESPHome 2026.9: two-second bus stalls
- **Symptom:** log messages such as `Clearing buffer of 1 bytes`, `parse failed`, `Stop waiting for response ... 2011974us` and `Poll refused by hub`. Flow updates paused for about 2 s.
- **Cause:** the meter emits a stray byte about 28 ms after each request. When it merges with the reply, the parse fails, and the new default `send_wait_time` of 2000 ms locks the bus.
- **Fix:** `send_wait_time: 400ms`, `turnaround_time: 100ms`.
- **Evidence:** 48/48 polls on time, a maximum gap of 1.05 s, and no parse failures ([docs/03](03-modbus-register-map.md)).

### 10. 5 V logic into a 3.3 V ESP32
- **Fix:** a 10k/10k/10k divider takes 2/3 of RO (5 V → 3.33 V) into GPIO38. The same divider brings the 0.5-4.5 V pressure signal into ADC1 (GPIO9).
- **Lesson:** keep analogue inputs on ADC1 on the ESP32-S3, and remember the 12 dB dead zone below about 150 mV.

---

## C. Home Assistant

### 11. Utility meters held 8.9 million litres
- **Symptom:** monthly and quarterly costs showed about £44,746.
- **Cause:** the meters had been fed by the broken total from issue 6.
- **Fix:** repoint them to the corrected `Mains Water Used`, then run `utility_meter.calibrate` with value 0. (`reset` only works with tariffs.)
- **Lesson:** history before the fix is not recoverable, so validate the source *before* building totals on it.

### 12. Two identical draws counted as one
- **Cause:** HA does not record a state change when the value is unchanged.
- **Fix:** the ESP32 adds +0.001 L when a draw volume equals the previous one.

### 13. Display precision greyed out
- **Cause:** YAML template sensors and utility meters had no `unique_id`.
- **Fix:** add `unique_id`. Templates then need a reload; utility meters need a full restart.

### 14. Phone alert never arrived
- **Cause:** `notify.send_message` targets a notify *entity*. The `notify.mobile_app_<phone>` *service* name is not an entity, and the action fails silently.
- **Fix:** target the phone's notify entity (find it under Developer tools → States → "notify.").

### 15. "Entity not found" for a new cost sensor
- **Cause:** a template was written but never added to `configuration.yaml`.
- **Lesson:** when handing over YAML, always say *where* it goes and which reload it needs.

### 16. Renamed entities left orphans behind
- **Cause:** renaming in ESPHome creates a new entity ID; the old one becomes `unavailable`.
- **Fix:** delete the orphans under Settings → Entities after each rename.

---

## D. Dashboard

| # | Problem | Fix |
|---|---|---|
| 17 | Small draws invisible on linear gauges | Gauge Card Pro with a square-root green zone (first third of the arc) |
| 18 | Gauges squashed in narrow columns | Sections `max_columns: 3`, section spans 2 + 1, theme `ha-view-sections-column-max-width: 800px` |
| 19 | ApexCharts `color_threshold` left all bars one colour | `apex_config.plotOptions.bar.colors.ranges` |
| 20 | Three filtered series with `transform` drew nothing | One series, coloured by value ranges |
| 21 | Status row never showed "Normal" | The error code state is `0.0`; use `numeric_state below: 0.5`, not a string match |
| 22 | Template Entity Row → "Configuration error" | The card wasn't installed; used built-in conditional rows instead |
| 23 | Gauge Card Pro not loading | Add the resource as a JavaScript module, then hard-refresh or reset the frontend cache |
| 24 | Logbook card couldn't limit rows or hide dots | Dropped. The draws bar chart conveys the same information |

---

## Top lessons

1. **Calibrate against reality, not the datasheet.** A clamp-on meter outside its rated pipe size can be stable yet badly wrong.
2. **Record the meter's raw output per event.** That single design choice turned noisy bucket tests into clean ratio data.
3. **Prove a register map against a value you can read on the display.**
4. **Nearly-right numbers hide offset and word-order bugs.**
5. **Read the release notes.** ESPHome 2026.9 changed Modbus timing defaults and retired options.
6. **Make suppression visible.** Every dead band and drift guard has a diagnostic flag.
7. **Change one variable at a time,** and log the health metrics (Q, M91) with every result.
