# ESP32 Voltage Logger: requirements

Version 0.1 (draft, 2026-11-09)

A reusable, multi-channel, multimeter-style voltage logger. Each channel is
galvanically isolated, reads AC and DC up to household mains voltage, and
reports to Home Assistant through ESPHome, so every measurement gets history
and graphs with no extra setup. It began in the Chiltrix CXI fan interposer
project (logging the unit's 15 V WH rail, which sits at mains potential), but
nothing in it is tied to that project.

## Decisions so far

| # | Decision | Date | By |
|---|----------|------|----|
| D-1 | Every channel is isolated on its own (not a shared ground). | 2026-11-09 | Scotty |
| D-2 | Logs AC and DC, up to household voltage (120/240 V AC). | 2026-11-09 | Scotty |
| D-3 | Reusable outside the interposer: own repo, generic firmware package. | 2026-11-09 | Scotty |
| D-4 | 4 channels. | 2026-11-09 | Scotty |
| D-5 | High and low input ranges. Every channel gets both an HV and an LV jack on a shared COM (no jumpers: they would sit on the mains side). Range selected in Home Assistant. | 2026-11-09 | Scotty (ranges), Claude (jack layout) |

## Open questions

- **Q-3 Current.** Out of scope for v1? (A clamp-on CT input could be added later.)

## Functional requirements

| ID | Requirement |
|----|-------------|
| FR-1 | 4 independent voltage channels (D-4). |
| FR-2 | Each channel reports DC (mean) and AC (RMS, DC removed). True RMS (AC+DC) is derived in Home Assistant or firmware. |
| FR-3 | HV range: at least 300 V RMS / 425 V peak, AC (45 to 65 Hz) or DC. |
| FR-4 | LV range: about 42 V peak, resolution 0.05 V or better after averaging (D-5). |
| FR-5 | Each channel is isolated from every other channel and from USB/ESP32 by a reinforced barrier. |
| FR-6 | Per channel, Home Assistant can select which range (HV or LV) is in use; the choice survives reboots. |
| FR-7 | Per channel calibration (gain and zero offset) set in YAML substitutions, no rebuild of hardware. |
| FR-8 | Channel names, update rates and calibration live in a small per-use device YAML; the shared package does not change between projects. |
| FR-9 | Report rate configurable, default 1 s. |
| FR-10 | Powered from a normal USB-C charger; no power bank needed because the inputs are isolated. |
| FR-11 | Expansion header: I2C (3.3 V) and one 1-Wire pin, for extra sensors (temperature, INA219/INA226 for low-voltage current). |
| FR-12 | Works without Home Assistant too: ESPHome web server shows live readings. |

## Hardware concept (per channel)

```
HV jack ──[fuse/flameproof R]──[4 x 1 MΩ HV string]──┐
LV jack ──[fuse/flameproof R]──[390 kΩ, 500 V]───────┤
                                                     ├── AMC3330 IN+ ──║── diff out ── op-amp (diff -> single-ended, 1.65 V mid)
COM jack ──────────────────────[8.45 kΩ]─────────────┘    AMC3330 IN-  ║            └── ESP32-S3 ADC1 pin
                                                          (integrated isolated DC-DC powers the HV side)
```

- **Isolated amplifier:** TI AMC3330 (reinforced isolation, ±1 V input, gain 2,
  integrated isolated DC-DC, so no separate isolated supply per channel).
  To confirm with a Digi-Key stock check before locking the design.
- **Scaling (nominal):** HV 425 V peak and LV 42 V peak both map to about
  ±0.9 V at the AMC3330 input, about ±1.08 V at the ADC around 1.65 V.
  Overall gain: HV about 395 V per ADC volt, LV about 39.3 V per ADC volt.
- **Fault case:** mains on the LV jack must not damage anything: the 390 kΩ
  then dissipates about 0.3 W, so it is a 1 W, 500 V rated part, and the
  AMC3330 input current stays well inside its limit.
- **ADC:** ESP32-S3 ADC1 (GPIO1 to GPIO4), calibrated per channel. A dedicated
  ADC is a possible later upgrade if the ESP32 ADC proves too noisy.
- **Board:** small 2-layer PCB (OSH Park), ESP32-S3-DevKitC-1 on headers.

## Safety requirements

| ID | Requirement |
|----|-------------|
| SR-1 | Aim at CAT II 300 V practice. This is a home-built, uncertified instrument; use it with that in mind. |
| SR-2 | Creepage across each channel's barrier and between channels at least 6.4 mm, with board slots under each AMC3330. |
| SR-3 | Shrouded 4 mm banana jacks and CAT II (or better) rated leads only. |
| SR-4 | Every HI input has a fuse or flameproof series resistor; HV divider split over several resistors so none sees more than its rated voltage. |
| SR-5 | Closed plastic enclosure; no exposed metal connected to any input. USB and expansion header are on the safe (ESP32) side only. |
| SR-6 | Connect leads with the circuit de-energised where possible. |

## Firmware

ESPHome package `firmware/voltage-logger.yaml`, included from a per-use device
file (see `firmware/example-device.yaml`). First cut uses ESPHome's `adc` plus
`ct_clamp` components; a small custom component that computes DC mean, AC RMS,
min and max in one sampling pass is planned.

## Change log

| Version | Date | Change |
|---------|------|--------|
| 0.1 | 2026-11-09 | First draft: D-1 to D-3, per-channel AMC3330 concept, open questions Q-1 to Q-3; D-4, D-5 settle Q-1 and Q-2. |
