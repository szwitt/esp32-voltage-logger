# ESP32 Voltage Logger

A reusable 4-channel, multimeter-style voltage logger. Every channel is
isolated on its own, reads **AC and DC up to household mains voltage**
(CH4 also logs **current and power** from a clamp-on CT), and
reports through **ESPHome to Home Assistant**, so every measurement gets
history and graphs right away.

Status: **design draft**. No hardware built yet.

## At a glance

- 4 channels, each with its own reinforced isolation barrier (TI AMC3330 isolated amplifier).
- CH1-CH3: HV jack (to 425 V peak, so 240 V AC with margin), LV jack (to about 42 V peak), shared COM.
  In HA: DC (mean), AC RMS, true RMS, and a Range select (HV/LV).
- CH4 power channel (ADE7953): HV jack + COM + CT socket. In HA: AC voltage, frequency, current, power, apparent power, power factor.
- ESP32-S3-DevKitC-1, USB-C powered from any charger (inputs are isolated, so no power bank).
- Expansion header: I2C plus 1-Wire for extra sensors.
- Built-in web page with live readings, for use without Home Assistant.

## Safety

This is a home-built, uncertified instrument that can connect to mains. It
follows CAT II 300 V practice (creepage, fused inputs, rated jacks and leads,
closed enclosure), but treat it with the care that implies. See
[docs/requirements.md](docs/requirements.md#safety-requirements).

## Layout

| Path | What |
|------|------|
| `docs/requirements.md` | Requirements, decisions, open questions, change log |
| `docs/project-log.md` | Running log of what happened and why |
| `firmware/voltage-logger.yaml` | Shared ESPHome package (don't edit per project) |
| `firmware/channel.yaml` | One voltage channel, included for CH1-CH3 |
| `firmware/power-channel.yaml` | CH4 power channel (ADE7953) |
| `firmware/example-device.yaml` | Copy this per logger: names and calibration |
| `hardware/` | Schematic and PCB (to come) |

## Using it in a project

1. Copy `firmware/example-device.yaml`, set `device_name`, channel names and calibration.
2. Add `secrets.yaml` (see `firmware/secrets.example.yaml`).
3. Flash from ESPHome Device Builder, adopt in Home Assistant.

## Origin

Started in the [Chiltrix CXI fan interposer](https://github.com/szwitt/chiltrix-cxi-fan-interposer)
project, to log the unit's 15 V rail that sits at mains potential.
