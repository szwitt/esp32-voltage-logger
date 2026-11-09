# ESP32 Voltage Logger: requirements

Version 0.2 (draft, 2026-11-09)

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
| D-5 | High and low input ranges. Every voltage channel (CH1-CH3) gets both an HV and an LV jack on a shared COM (no jumpers: they would sit on the mains side). Range selected in Home Assistant. | 2026-11-09 | Scotty (ranges), Claude (jack layout) |
| D-6 | CH4 is a power channel (voltage, current, power) using one energy-meter chip; CH1-CH3 stay voltage-only. ADI ADE7953 chosen over the Cirrus CS5460A (still in production, ESPHome driver, I2C so a 2-wire isolator, two current inputs). | 2026-11-09 | Scotty (power channel), Claude (chip) |
| D-7 | CH4 current comes from a clamp-on split-core CT only (AC). No in-line shunt. | 2026-11-09 | Scotty |
| D-8 | CH4 is sized for small loads: SCT-013-000 current-output CT (100 A : 50 mA) with a 33 Ω burden, about 21 A RMS full scale. For very small loads, loop the wire through the clamp N times and divide by N (set in calibration). | 2026-11-09 | Scotty (small loads), Claude (CT and burden) |

## Notes

- DC current is not on the board; for low-voltage projects add an INA226 on the expansion header.

## Functional requirements

| ID | Requirement |
|----|-------------|
| FR-1 | 4 independent channels (D-4): CH1-CH3 voltage, CH4 power (D-6). |
| FR-2 | CH1-CH3 each report DC (mean) and AC (RMS, DC removed). True RMS (AC+DC) is derived in Home Assistant or firmware. |
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
| FR-13 | CH4 reports AC voltage RMS, frequency, CT current RMS, active power, apparent power and power factor (D-6, D-7). CH4 voltage is AC only (the chip's high-pass filter is on). |
| FR-14 | CH4 voltage: one HV jack + COM, at least 300 V RMS. The chip's 24-bit converter resolves low voltages on the same range, so no LV jack. |
| FR-15 | CH4 current: 3.5 mm socket for a split-core CT, 33 Ω burden on the board (D-8). |
| FR-16 | CH4 calibration: voltage, current and power multipliers in YAML substitutions. |

## Hardware concept: voltage channels CH1-CH3

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
- **ADC:** ESP32-S3 ADC1 (GPIO1, GPIO2, GPIO5, GPIO6; GPIO3 is a strapping pin), calibrated per channel. A dedicated
  ADC is a possible later upgrade if the ESP32 ADC proves too noisy.
- **Board:** small 2-layer PCB (OSH Park), ESP32-S3-DevKitC-1 on headers.

## Hardware concept: power channel CH4

```
HV jack ──[fuse/flameproof R]──[4 x 240 kΩ HV string]──┬── ADE7953 V+   (±0.5 V pk max; 425 V pk -> about 0.44 V)
COM jack ───────────────────────[1 kΩ]─────────────────┴── ADE7953 V-, AGND
CT socket (3.5 mm) ── 33 Ω burden (about 21 A RMS full scale) ── ADE7953 IAP/IAN    (current input B left unpopulated)
ADE7953 I2C (0x38) ──║ ISO1640 (reinforced I2C isolator) ║── ESP32 I2C GPIO8/9
3.3 V for ADE7953    ──║ UCC12050 (reinforced isolated DC-DC) ║── ESP32 5 V
3.58 MHz crystal on the ADE7953
```

- Parts (ADE7953, ISO1640, UCC12050, CT) to confirm with a Digi-Key stock check.
- ESPHome's `ade7953_i2c` driver uses fixed scale factors; FR-16 multipliers correct them.
- The CT socket is on CH4's isolated (hazardous) side: it must be recessed so no
  metal can be touched, and the CT plug goes in before the voltage leads are
  connected. Connect CH4 COM to neutral where possible.

## Safety requirements

| ID | Requirement |
|----|-------------|
| SR-1 | Aim at CAT II 300 V practice. This is a home-built, uncertified instrument; use it with that in mind. |
| SR-2 | Creepage across each channel's barrier and between channels at least 6.4 mm, with board slots under each AMC3330, ISO1640 and UCC12050. |
| SR-3 | Shrouded 4 mm banana jacks and CAT II (or better) rated leads only. |
| SR-4 | Every HI input has a fuse or flameproof series resistor; HV divider split over several resistors so none sees more than its rated voltage. |
| SR-5 | Closed plastic enclosure; no exposed metal connected to any input. USB and expansion header are on the safe (ESP32) side only. |
| SR-6 | Connect leads with the circuit de-energised where possible. |

## Firmware

ESPHome package `firmware/voltage-logger.yaml`, included from a per-use device
file (see `firmware/example-device.yaml`). First cut uses ESPHome's `adc` plus
`ct_clamp` components for CH1-CH3 and `ade7953_i2c` for CH4
(`firmware/power-channel.yaml`); a small custom component that computes DC mean, AC RMS,
min and max in one sampling pass is planned.

## Change log

| Version | Date | Change |
|---------|------|--------|
| 0.1 | 2026-11-09 | First draft: D-1 to D-3, per-channel AMC3330 concept, open questions Q-1 to Q-3; D-4, D-5 settle Q-1 and Q-2. |
| 0.2 | 2026-11-09 | D-6 CH4 power channel with ADE7953 (settles Q-3), D-7 CT-only current. FR-13 to FR-16, CH4 hardware concept. D-8 CT for small loads (33 Ω burden). |
