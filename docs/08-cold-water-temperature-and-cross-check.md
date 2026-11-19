# 08 · Cold Water, the Travel-Time Window and the Smart-Meter Cross-Check

[← Back to README](../README.md)

**Status (16 Nov 2026): the corrected meter reads well below the water company's smart meter as the mains water gets colder. Diagnosis: the install sits near the edge of the meter's travel-time window, and every draw brings the coldest water. Plan: a one-off spacing and pipe-settings check, then a correction fitted against the travel-time ratio and the daily cross-check.**

## 1. Cross-check against the smart meter

A cross-check package (daily TUF totals at 23:59:50 with a 14-day history attribute, and the % difference for the latest date the water company has published) compares this meter with the utility's own smart meter.

| Day | This meter | Smart meter | Difference |
|---|---|---|---|
| 12 Nov | ~480 L | 958 L | **~50 % low** |

The corrected daily totals also fell from ~600 to ~300 L/day over the period as the mains water cooled, with no matching change in household use. Earlier shortfalls were 12-42 %.

## 2. What the meter's own diagnostics show

| Condition | Travel-time ratio | Note |
|---|---|---|
| At rest (room-temperature water in the pipe) | ~102.7 % | Inside the meter's ~100 ± 3 % window |
| During every cold draw (15:30, 18:40, 22:20 on 14-15 Nov) | **108-109 %**, decaying over ~1 h | Outside the window |
| Meter's own sound-speed estimate, cold | 1,367 m/s | Below real water (~1,425 m/s at 5 °C) |

Flow is only measured during draws, which is exactly when the water is coldest and the ratio furthest out of its window.

## 3. Why temperature matters, and what an NTC can and can't fix

Colder water changes the speed of sound, which changes the refraction angle through the pipe wall, so the beam lands off the receiving transducer. That has two effects:

1. **Signal degradation** (ratio out of window, lower quality): physical, so a temperature measurement **can't** fix it. Only the transducer geometry can.
2. **Calculation error** (the meter's assumed sound speed): may be correctable in software **if** it is consistent against temperature.

Re-spacing for each season is not normal practice: a 5-20 °C mains shift changes the sound speed by about 4 %, a few percent error on a well set-up meter. The size of the error here suggests the install sits near the edge of its window to begin with.

## 4. Plan (in order)

1. One-off check: the meter's recommended spacing vs the actual spacing; pipe settings (outside diameter, wall thickness, copper, no lining); re-grease and re-seat the transducers; the low-flow cut-off (small draws lost as zero). Aim to centre the window (warm ~98-99 %, cold ~104-105 %).
2. Then fit a correction against the **travel-time ratio** (itself a temperature proxy measured during every draw), checked against the daily cross-check.
3. A strap-on NTC on the cold main as an independent check (also the cold-feed value for the heating repo's hot water estimate).

## 5. Mains pressure under flow

The README's "about 0.07 bar sag" applies to small draws. At shower flows (7-9 L/min) the mains sags **~0.5-0.6 bar** (≈3.0 → ≈2.4 bar), and recovers to ≈2.9 bar at rest.

## 6. Packages added (13 Nov)

- [`config/homeassistant/water_meter_crosscheck.yaml`](../config/homeassistant/water_meter_crosscheck.yaml): "TUF daily totals" (recorded 23:59:50, 14-day history) and "Water meter cross-check" (% difference for the date of the smart meter's latest reading, which arrives ~4 days late). The utility integration's entity IDs contain the account reference; it is replaced with `<yw_account_ref>` here.
- [`config/homeassistant/water_tariff.yaml`](../config/homeassistant/water_tariff.yaml): tariff helpers (water and sewerage £/m³, return factor, standing charges), defaults automation and an April reminder. Every cost sensor reads these, so a tariff change is one edit. The 2026/27 metered rates were confirmed from the bill page; the supplier's own app/integration costs used last year's water rate and foul-only sewerage, so they under-state.

## 7. Shower sub-metering fixes (12-13 Nov)

- **Shower running:** motion starts it, flow keeps it running (the motion sensor lost people standing still in the shower: a 6 L "shower").
- **Humidity confirmation** threshold +10 → +6 points; running stays on only with confirmation, recent motion or within the first 8 min; usage recorded only when confirmed; status sensor (Off / Starting (awaiting humidity) / Running (confirmed)); back-to-back showers keep separate records with the earlier dry-room humidity baseline.
- First full capture read about half the true flow, consistent with this meter's cold-water shortfall (§1-3).

## 8. Addendum (18-19 Nov 2026): the cross-check fixed, and five days of figures

**Package fix (18 Nov, live):** the cross-check returned the *text* "unknown" when there was nothing to compare, which Home Assistant logged as an invalid state on every reload. It now returns no value instead; the smart meter's `reading_date` is cut to `YYYY-MM-DD` before matching; and the nightly TUF snapshot skips a night with no numeric total instead of recording 0 L. Published: [`config/homeassistant/water_meter_crosscheck.yaml`](../config/homeassistant/water_meter_crosscheck.yaml) (account reference replaced with `<yw_account_ref>`).

**TUF as a share of the smart meter's daily figure:**

| Day | TUF ÷ smart meter |
|---|---|
| 13 Nov | 43 % |
| 14 Nov | 37 % |
| 15 Nov | 43 % |
| 16 Nov | 39 % (cross-check −61.0 %) |
| 17 Nov | 52 % (cross-check −48.1 %) |

**17 Nov in detail (from the history):** the 20 L/min cap engaged for 36 s in total; the drift guard never fired; dropouts lasted at most 6 s. The meter's reverse total grew by 163 L, but that is **at-rest jitter around zero**, not lost water (it does make the meter's own today/month totals meaningless). Every draw was counted, but **under-read**, while the travel-time ratio sat at 104-108 % during the morning showers.

**Conclusion unchanged:** the shortfall is a **scale** error that grows as the water cools (§1-3), not missed draws. **Next:** a cold-water bucket test (10 L, three flow rates × three fills, noting the meter's draw volume and raw totaliser each time) to refit the correction table; then the one-off spacing and pipe-settings check; then the temperature/ratio-based correction.
