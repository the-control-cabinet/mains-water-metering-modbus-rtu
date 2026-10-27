# 02 · Meter Installation and Configuration

[← Back to README](../README.md)

## Installation geometry

```
            ↑ to sink tee and the rest of the house
            │
        [STOPCOCK]
            │   a few cm clear
          ┌─┴─┐
          │DN │  ← TS-2 "DN" (downstream), cable end pointing UP
          └─┬─┘
            │   gap (pipe clip to wall sits here)
          ┌─┴─┐
          │UP │  ← TS-2 "UP" (upstream), cable end pointing DOWN
          └─┬─┘
            │   about 20 cm vertical run in total
     ───────┘   90° bend
   mains in →
```

| Item | Value |
|---|---|
| Pipe | 22 mm copper, OD measured 21.75 mm, wall assumed 0.9 mm |
| Flow direction | Upward through the vertical run |
| Mounting | **W method**: both transducers on the same side of the pipe, in line along the axis, on the free side away from the wall |
| Orientation | The oval (emitting) faces point toward each other across the gap |
| Spacing (M25) | **24.41 mm**, measured between the **inner edges of the transducer casings** (not oval to oval) |
| Couplant | Vaseline now, silicone grease planned |
| Straight runs | Far shorter than the textbook 10D upstream / 5D downstream. A bend lies about 20 cm below and the stopcock just above |

### Why W, not V or Z?

- **V method:** on 22 mm thin-wall copper it calculates a **negative** spacing, so it is physically impossible.
- **Z method:** needs opposite-side mounting. This was not tried; the wall position makes it awkward.
- **W method:** the "small pipe" option. Its longer acoustic path gives more time difference to work with.

## Menu settings (as commissioned)

| Window | Setting | Value | Why |
|---|---|---|---|
| M11 | Pipe outer diameter | 22 mm (measured 21.75) | |
| M12 | Wall thickness | 0.9 mm | 0.7 and 1.2 were tried; no fix for the under-read |
| M14 | Pipe material | Copper | |
| M20 | Fluid | Water (general) | |
| M23 | Transducer type | **Standard S** | "Clamp-on TS-2" (option 19) tested: worse, 37 % vs 48 % |
| M24 | Mounting method | W (3) | See above |
| M25 | Transducer spacing | 24.41 mm (computed by the meter) | |
| M31 | Flow display unit | L/min | The Modbus flow register is **always m³/h** |
| M32 | Totaliser unit | L (multiplier ×1) | Totaliser registers follow M31/M32 |
| M40 | Damping | 2 s | Faster response for short draws |
| M41 | Low-flow cut-off | 0.02 m/s (default 0.03) | Lets slow draws register; drift is handled in firmware |
| M42 | Set zero | Done with **true no flow** | Never set zero with water moving |
| M43 | Reset to factory zero | Used before re-zeroing | |
| M44 | Manual zero offset | 0 | It is a flow bias on this firmware; must be 0 |
| M45 | Scale factor | **1.5** (meter maximum) | Further correction is done in the ESP32 |
| M29 | Empty-pipe signal threshold | 39 | Signal is about 76, so wide margin |
| M83 | Offline compensation (auto-amend) | Off | The manual advises against it |
| M26 | **Solidify / save settings** | Done | Without this, zero and settings are lost on power-down |

## Diagnostic windows worth knowing

| Window | Shows | Healthy on this install |
|---|---|---|
| M90 | Signal strength (up/down) and quality Q | Strength about 76, **Q 83-88 %**. The manual minimum is Q > 60; field experience suggests > 85 |
| M91 | Travel-time ratio | 100 ± 3 % ideal; **102.9-104.3 %** here. Swings of 80-115 % mean a transducer is moving |
| M92 | Fluid sound speed | About 1,430-1,440 m/s at rest |
| M93 | Total transit time and delta time | Use for before/after comparisons |
| M94 | Reynolds number and pipe factor | Factor 0.75 laminar, 0.80 at Re about 3000, 0.925 at Re about 8400 |
| M00 / M02 / M03 | Net / positive / negative totals | NET = POS here; NEG = 0 |

### Sound speed is not a thermometer

The "sound speed" in M92 is **derived from the measured travel times and the configured geometry**, not measured independently. During a shower it drops from about 1,430 to about 1,390 m/s. That is the ratio shifting with flow, not the water getting colder. A real pipe NTC is on the roadmap.

## Commissioning sequence that worked

1. Clean the pipe to bright copper where the transducers sit, then mark the positions with a pen.
2. Enter M11-M24 and read the spacing from M25.
3. Fit the transducers with couplant and straps at that spacing.
4. Shuffle each transducer by 1-2 mm until **Q > 85** and **M91 is within 100 ± 3-4 %** at the same time.
5. Turn the water fully off downstream and wait 2 minutes. Then **M43**, then **M42**.
6. Run **M26** to store everything.
7. Photograph the settings and the pen marks so the installation is repeatable.
8. Calibrate against buckets ([docs/04](04-calibration-and-testing.md)).
