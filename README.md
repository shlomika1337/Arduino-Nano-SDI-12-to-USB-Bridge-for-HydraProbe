# HydraProbe SDI-12 Reader (Arduino Nano)

Read a Stevens HydraProbe soil sensor over the SDI-12 bus with an Arduino Nano (ATmega328P) and print the decoded measurements to the serial monitor.

The sketch wakes the sensor, issues a measurement command, collects the values across the sensor's `D` responses, and parses them into a fixed-size float array.

## Hardware

| Part | Notes |
|------|-------|
| Arduino Nano (ATmega328P) | A clone is fine; powered over USB from the PC |
| Stevens HydraProbe (SDI-12) | 3-wire: red = power, black = ground, blue = data |
| 12 V DC adapter | Powers the probe. **The probe needs ~12 V — it will not work on 5 V.** |

### Wiring

The Nano is powered over USB. The probe is powered from the 12 V adapter. The two **must share a common ground** — without it the SDI-12 data line has no voltage reference and the sensor stays silent.

```
HydraProbe red   ── 12V adapter (+)
HydraProbe black ──┬─ 12V adapter (−)
                   └─ Nano GND          <-- the easy-to-forget wire
HydraProbe blue  ── Nano D7             (data; any digital pin except D0/D1)
Nano             ── PC over USB
```

Two wires run between the Nano and the probe: **data** (blue → D7) and **ground** (black → GND). The PC connection is USB only.

## Software

Requires the [Arduino-SDI-12](https://github.com/EnviroDIY/Arduino-SDI-12) library (install via the Arduino Library Manager: search "SDI-12"). SDI-12 is a single-wire, half-duplex, inverted-logic protocol, so this library handles the bit-level timing — a plain `Serial` read will not work.

Set the Serial Monitor to **115200 baud** and a line ending of **Newline**.

## Usage

1. Wire everything as above and double-check the common ground.
2. Open the sketch, confirm `SDI12_DATA_PIN` matches your data pin (default `7`).
3. Upload, open the Serial Monitor at 115200 baud.
4. On boot the sketch sends `0I!` and prints the sensor identification.
5. Press **Enter** in the monitor to trigger a measurement cycle.

### Example output

```
0I!
012STEVENSW0560126.302GSN00303589
0M!
00029
0D0!
00+0.000+0.000+25.4
0D1!
0+77.7+0.000+1.459
0D2!
0+0.046-0.005+0.032
```

## Understanding the responses

**Identification (`0I!`)** — `0` (address), SDI-12 v`12`, vendor `STEVENSW`, model `0560`, plus version and serial number.

**Measurement (`0M!`) → `00029`** — decodes as `a tt n`:
- `0` — sensor address
- `002` — seconds until data is ready
- `9` — number of values available

**Data (`0D0!`, `0D1!`, ...)** — each response is the address followed by sign-prefixed values (`+`/`-`). SDI-12 limits the length of each `D` response, so the 9 values arrive in chunks across `D0!`–`D2!`. The remaining `D` commands return nothing.

Values sitting in open air read near-zero moisture with a temperature around room temperature and a dielectric permittivity that jumps sharply once the tines are in water — a quick distilled-water dip is the standard sanity check.

## Parsing

Because every SDI-12 value carries its own `+`/`-` sign, the values can be tokenized on the sign characters, which also skips the leading address (everything before the first sign is ignored). The parser appends across multiple `D` responses into one array, stopping once the expected number of values is collected.

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| No response to any command (silence, or only stray `!` echoes) | Missing common ground between Nano and 12 V adapter — check first |
| Still silent after grounding | No 12 V at the probe (measure red-to-black at the probe end); check adapter polarity |
| `0I!` works but wrong/no data | Sensor at a non-default address — query with `?!` to discover it |
| Garbled characters | Baud rate mismatch — set the monitor to 115200 |

## License

The SDI-12 example code this project is based on is published by Stroud Water Research Center under the BSD-3 license.
