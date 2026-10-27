# 01 · Hardware and Wiring

[← Back to README](../README.md)

## Bill of materials

| Item | Part | Notes |
|---|---|---|
| Flow meter | TUF-2000M ultrasonic module (keypad and display) | Powered from 24 V DC. Modbus RTU slave over RS485 |
| Transducers | 2 × TS-2 clamp-on (small) | Listed as DN25-100. Body about 45 × 25 × 32 mm |
| Couplant | Vaseline (temporary) → silicone grease | Silicone grease is more stable long-term |
| Fixings | Metal straps over each transducer | |
| RS485 interface | DollaTek MAX485 TTL-RS485 module (5 V) | DE and RE tied together |
| Microcontroller | ESP32-S3-DevKitC-1, 16 MB flash | ESPHome, ESP-IDF framework |
| Pressure sensor | DFRobot SEN0257 Gravity water pressure | 0-1.6 MPa (0-16 bar), G1/4, IP68, 5 V supply, 0.5-4.5 V output |
| Pressure fitting | 1/2" × 1/4" × 1/2" BSP tee | Dowty (bonded) washer on the BSP thread. Watch for BSP vs NPT mismatch |
| Dividers | 10 kΩ resistors, Wago lever connectors | Two dividers, one for RX and one for the pressure signal |

## Pin map (ESP32-S3)

| GPIO | Function | Notes |
|---|---|---|
| 41 | UART TX → MAX485 DI | |
| 38 | UART RX ← MAX485 RO (through a 2/3 divider) | Never connect RO directly. It is 5 V logic |
| 2 | MAX485 DE + RE (`flow_control_pin`) | High = transmit |
| 21 | Divider bridge pad (not read) | Used only as a solder or terminal point |
| 9 | ADC1: pressure signal (through a 2/3 divider) | 12 dB attenuation |
| 12 | Divider bridge pad (not read) | |
| 4 or 5 | *Planned:* pipe NTC (ADC1) | See the roadmap |

> **Why ADC1?** On the ESP32-S3, ADC2 is unusable while Wi-Fi is active. Keep analogue inputs on GPIO1-10.

## RS485 link

```
TUF-2000M 485+  ───────── A  ┐
TUF-2000M 485-  ───────── B  │ MAX485 module (VCC 5 V, GND common with the ESP32)
                             │
ESP32 GPIO41 ──────────── DI │
ESP32 GPIO2  ──────────── DE+RE (bridged)
MAX485 RO ── 10k ──┬── 10k ──┬── 10k ── GND
                   │         └── GPIO21 (bridge pad only)
                   └── GPIO38 (UART RX)
```

The read node sits at 20k/30k = **2/3 of RO**, so 5 V becomes about 3.33 V.

If the meter never replies, **swap A and B first**. The labelling is inconsistent between vendors.

Meter comms settings: **M62** = 9600 baud, no parity, 8 data bits, 1 stop bit. **M63** = Modbus RTU. **M46** = network address 1.

## Pressure sensor

```
SEN0257 red    ── 5 V
SEN0257 black  ── GND
SEN0257 yellow ── 10k ──┬── 10k ──┬── 10k ── GND
                        │         └── GPIO12 (bridge pad only)
                        └── GPIO9 (ADC1)
```

- **Pin voltage** = 2/3 × sensor voltage. The firmware divides by 0.6667 to get the sensor voltage back.
- **Pressure (bar)** = (V_sensor − 0.454) / 4.0 × 16.
  - The 0.454 V zero was measured with the sensor open to air.
  - The datasheet gives 0.5 V nominal.
- **12 dB attenuation** has a non-linear dead zone below about 150 mV at the pin. A 0.5-4.5 V sensor sits well above it.
- **Changing the divider:** if the lower leg is ever changed to 22 kΩ (giving 44/54 = 0.815), update the `/ 0.6667` in the YAML.
- **Span check outstanding:** the span has not yet been checked against a mechanical gauge (see the [roadmap](07-roadmap.md)).
- **Location:** teed into the cold supply at the sink isolation valve, downstream of the stopcock.

## Board firmware notes

- **Bluetooth:** passive BLE scanning only, used by the Bermuda room-presence integration. Settings are interval 320 ms, window 30 ms, `active: false`. It shares the radio happily with the Modbus work.
- **API:** encrypted. All credentials are `!secret` (see `config/esphome/secrets.example.yaml`).
- **Fallback hotspot:** password must be 8 characters or more.
- **Flashing:** USB for the first flash, OTA afterwards. Rotating the API key later needs one USB flash.
