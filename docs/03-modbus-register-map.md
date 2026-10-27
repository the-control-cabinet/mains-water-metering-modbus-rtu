# 03 · Modbus Register Map and ESPHome Polling

[← Back to README](../README.md)

## The two rules that matter

1. **Address = manual register number − 1.** The TUF-2000M manual numbers registers from 0001. The manual's own example reads REG0001-REG0010 with `01 03 00 00 00 0A`, which starts at PDU address 0.
2. **32-bit values are low word first.** In ESPHome that means `FP32_R` for REAL4 floats and `U_DWORD_R` for LONGs.

Get either wrong and the values *look* plausible, which is the dangerous part (see [issues 4 and 5](06-issues-and-lessons-learned.md)).

## Registers used

| Manual reg | ESPHome `address` | Type | Content | Poller |
|---|---|---|---|---|
| 0001-0002 | 0 | FP32_R | Flow rate, **m³/h always** (regardless of M31) | fast 1 s |
| 0005-0006 | 4 | FP32_R | Flow velocity, m/s | slow 60 s |
| 0007-0008 | 6 | FP32_R | Fluid sound speed, m/s | slow |
| 0072 | 71 | U_WORD | Error code (0 = OK) | slow |
| 0092 | 91 | U_WORD, mask `0x00FF` | Signal quality, % (low byte) | slow |
| 0093 | 92 | U_WORD × 0.0244 | Upstream strength | slow |
| 0094 | 93 | U_WORD × 0.0244 | Downstream strength | slow |
| 0097-0098 | 96 | FP32_R | Travel-time ratio, % | slow |
| 0099-0100 | 98 | FP32_R | Reynolds number | slow |
| 0101-0102 | 100 | FP32_R | Pipe (profile) factor | slow |
| 0103-0104 | 102 | U_DWORD_R | Working timer, s | slow |
| 0115-0116 | 114 | FP32_R | Positive total (float, in the totaliser unit) | slow |
| 0117-0118 | 116 | FP32_R | Negative total | slow |
| 0125-0126 | 124 | FP32_R | Flow today | slow |
| 0127-0128 | 126 | FP32_R | Flow this month | slow |

Registers not used:

- **0033-0036:** PT100 temperatures. Not wired, so the values are meaningless.
- **0009-0012:** integer + fraction totals.
- **0089:** current output.
- **1437-1439:** unit registers.

### Notes

- **Strength scale.** The manual says 0-2047, but on this firmware a raw count of about **3,141** corresponds to a display of **76.6**, so the factor is **0.0244**.
- **Totaliser units.** The totaliser registers use the **display** units (M31/M32 and the multiplier). The flow register does not. It is fixed at m³/h.
- **Checking the total.** The float at address 114 read 0.632 m³ on the check date, exactly matching the meter's own M02 total.

## ESPHome 2026.9 polling design

```yaml
modbus:
  id: modbus1
  uart_id: modbus_uart
  send_wait_time: 400ms     # default 2000 ms: one bad frame = 2 s of dead bus
  turnaround_time: 100ms

modbus_controller:
  - id: tuf2000m            # fast: flow only, 1 s
    address: 1
    modbus_id: modbus1
    update_interval: 1s
  - id: tuf2000m_slow       # slow: all "Meter Stats", 60 s
    address: 1
    modbus_id: modbus1
    update_interval: 60s
```

Points specific to 2026.9:

- **`skip_updates` is retired.** It no longer slows individual sensors. The supported pattern is a **second controller at the same slave address** with a longer interval.
- **`command_throttle` is replaced by `turnaround_time`.**
- **`register_count` is replaced by `reuse_previous_range`.**
- **Address 0 is valid for a register**, so `address: 0` on the flow sensor is fine. Only a *device* address of 0 is rejected.

### The stray-byte behaviour

About 28 ms after each request the bus carries **one stray byte**, and the real reply follows 14-50 ms later. Usually ESPHome just logs:

```
Clearing buffer of 1 bytes - timeout after partial response
```

Occasionally the stray byte merges with the reply, giving `parse failed`. With default settings the hub then stalls for `send_wait_time` (2 s) and refuses polls:

```
Stop waiting for response ... 2011974us
Frame already active for 1 with 2 requests pending, refused
Poll refused by hub for range 0x0
```

**Results after the timing change:**

| Measure | Result |
|---|---|
| Successful flow polls | 48 over about 47 s |
| Longest gap between polls | 1.05 s |
| Longest gap during the 60 s stats burst | 0.84 s |
| Stray-byte warnings | About one per 10 s (was one every 2 s) |
| Parse failures / refused polls | None |

The cause of the stray byte is not proven. It may be a driver-enable glitch on the meter, or line bias; fitting bias resistors is an untested option. Related ESPHome reports: issues 20780, 17180, 19972 and PR 11781.
