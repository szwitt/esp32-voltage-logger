# ESP32 Voltage Loggers

Two small, reusable, multimeter-style loggers that report through
**ESPHome to Home Assistant**, so every measurement gets history and graphs
right away. Through-hole parts and pre-built modules only.

Status: **design draft**. No hardware built yet.

## Logger M (mains)

- 2 circuits, each a PZEM-004T v3.0 module with a split-core 100 A CT.
- Per circuit in HA: AC voltage, current, power, energy, frequency, power factor (80-260 V AC).
- USB-powered from any charger (the PZEM serial ports are isolated).

## Logger L (low voltage)

- 4 channels, each an Adafruit INA228 breakout: 0-85 V DC plus optional DC current, mean/min/max per interval.
- F1, F2: optocoupled frequency/state inputs (tach, contacts), 3-30 V, up to 5 kHz.
- All channels share one COM; runs from a power bank so it can float on a circuit's reference.

Both have a built-in web page with live readings, for use without Home Assistant.

## Safety

These are home-built, uncertified instruments, and both can end up at mains
potential (Logger M always, Logger L when floating on a mains-referenced
circuit). Shrouded jacks, rated leads, closed enclosures. See
[docs/requirements.md](docs/requirements.md#safety-requirements).

## Layout

| Path | What |
|------|------|
| `docs/requirements.md` | Requirements, decisions, change log |
| `docs/project-log.md` | Running log of what happened and why |
| `firmware/common.yaml` | Shared ESP32-S3 base |
| `firmware/mains-logger.yaml`, `mains-channel.yaml` | Logger M package |
| `firmware/lv-logger.yaml`, `lv-channel.yaml`, `freq-inputs.yaml` | Logger L package |
| `firmware/example-mains.yaml`, `example-lv.yaml` | Copy these per logger: names, units, calibration |
| `hardware/digikey-stock-check.csv` | Digi-Key parts for both loggers |

## Using one in a project

1. Copy `firmware/example-mains.yaml` or `firmware/example-lv.yaml`, set `device_name` and channel names.
2. Add `secrets.yaml` (see `firmware/secrets.example.yaml`).
3. Flash from ESPHome Device Builder, adopt in Home Assistant.

## Origin

Started in the [Chiltrix CXI fan interposer](https://github.com/szwitt/chiltrix-cxi-fan-interposer)
project, to log the unit's 15 V rail that sits at mains potential.
