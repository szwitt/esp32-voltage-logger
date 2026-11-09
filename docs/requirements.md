# ESP32 Voltage Loggers: requirements

Version 0.5 (draft, 2026-11-09)

Two small, reusable, multimeter-style loggers that report through ESPHome to
Home Assistant, so every measurement gets history and graphs with no extra
setup. Both use an ESP32-S3-DevKitC-1 and through-hole parts or pre-built
modules only (no surface-mount soldering).

- **Logger M (mains):** AC mains voltage, current and power, 2 circuits.
- **Logger L (low voltage):** 4 DC voltage/current channels up to 85 V,
  2 frequency/state inputs, min/max capture.

They began in the Chiltrix CXI fan interposer project (logging the unit's
15 V WH rail, which sits at mains potential), but nothing in them is tied to
that project.

## Decisions

| # | Decision | Date | By |
|---|----------|------|----|
| D-1 | *(Superseded by D-12.)* Every channel isolated on its own. | 2026-11-09 | Scotty |
| D-2 | Log AC and DC up to household voltage (120/240 V AC). Now split across the two loggers (D-12). | 2026-11-09 | Scotty |
| D-3 | Reusable outside the interposer: own repo, generic firmware packages. | 2026-11-09 | Scotty |
| D-4 to D-8 | *(Superseded.)* 4-channel combined board, HV/LV jacks per channel, ADE7953 power channel, SCT-013 CT sizing. See change log 0.1 to 0.3. | 2026-11-09 | Scotty |
| D-9 | All through-hole or pre-built modules; no SMD soldering (no tooling). | 2026-11-09 | Scotty |
| D-10 | Mains power measurement with ready-made PZEM-004T v3.0 modules (100 A, split-core CT). | 2026-11-09 | Scotty |
| D-11 | Requirements scrub against the interposer: frequency/state inputs and min/max per interval are needed; waveform, PWM duty and sub-millisecond work stay with the oscilloscope. | 2026-11-09 | Scotty (scrub), Claude (findings) |
| D-12 | Split into two loggers (M and L) instead of one combined board with per-channel isolation. Under half the cost (about $85-115 vs about $235) and mostly pre-built modules. Accepted losses: no per-channel isolation on L (shared COM), no isolated high-voltage DC (for example a 160 V motor bus). | 2026-11-09 | Scotty |

## Logger M (mains)

| ID | Requirement |
|----|-------------|
| FR-M1 | 2 mains circuits (M1, M2), each a PZEM-004T v3.0 (100 A) on its own ESP32 UART (UART1 GPIO17/18, UART2 GPIO15/16). |
| FR-M2 | Per circuit: AC voltage, current, active power, energy, frequency, power factor. 80 to 260 V AC, 0.02 to 100 A, 0.1 V / 0.001 A resolution. |
| FR-M3 | Current from the PZEM's own split-core CT (clamp on one conductor; never break the circuit). |
| FR-M4 | Voltage sense leads in through shrouded 4 mm jacks (Pomona 72930, IEC 61010), one red/black pair per circuit, fused inside. |
| FR-M5 | Powered from a normal USB charger: each PZEM's serial port is optocoupled, so the ESP32 side is isolated from mains. |
| FR-M6 | Optional per-circuit calibration trims (voltage, current, power multipliers) in YAML. |

Notes:
- AC mains only. It reads nothing below 80 V or on DC.
- A third circuit is possible on a shared UART with a different Modbus
  address (set once with a PZEM address tool).

## Logger L (low voltage)

| ID | Requirement |
|----|-------------|
| FR-L1 | 4 channels (L1 to L4), each an Adafruit INA228 breakout on I2C (0x40, 0x41, 0x44, 0x45): bus voltage 0 to 85 V DC referenced to the logger's COM, 20-bit. |
| FR-L2 | Each channel can also measure DC current (up to 10 A) through the breakout's 15 mΩ shunt; for voltage only, the channel jack feeds VIN- with VIN+ bridged. |
| FR-L3 | Two optocoupled frequency/state inputs F1, F2 (LTV-827, 10 kΩ + 1N4148 per input) into GPIO10/11: 3 to 30 V signals, up to 5 kHz, reported as frequency with a per-input scale (Hz, rpm) and as on/off state. |
| FR-L4 | Each channel reports the mean, minimum and maximum over each report interval (default: sampled every 100 ms, reported every 1 s), so ~1 s dips show up. |
| FR-L5 | All channels share one COM jack. Inputs on shrouded 4 mm jacks (Pomona 72930): red L1-L4, blue F1-F2, black COM. |
| FR-L6 | Runs from a USB power bank inside the enclosure, so the whole logger can float with COM on the circuit's reference (for example BK at mains potential, as on the interposer). A 20 Ah bank lasts roughly a week. |

Notes:
- DC only. Low-voltage AC (for example 24 VAC thermostat wiring) is not
  covered by either logger yet.
- Two circuits with unrelated grounds can't be logged at the same time on
  one Logger L (shared COM).

## Common requirements

| ID | Requirement |
|----|-------------|
| FR-C1 | ESPHome packages, included from a short per-use device YAML that sets names, units and calibration; the packages don't change between projects. |
| FR-C2 | Report rate configurable, default 1 s. |
| FR-C3 | Works without Home Assistant: ESPHome web server shows live readings. |
| FR-C4 | Through-hole parts and pre-built modules only (D-9); perfboard or a small 2-layer PCB. |

## Safety requirements

| ID | Requirement |
|----|-------------|
| SR-1 | Home-built, uncertified instruments; aim at CAT II 300 V practice for anything that touches mains. |
| SR-2 | Shrouded 4 mm jacks (IEC 61010) and CAT II (or better) sheathed leads only. Wiring from jacks to boards 600 V rated. |
| SR-3 | Closed plastic enclosures; no exposed metal connected to any input. |
| SR-4 | Logger M: keep the PZEM mains terminals and the CT leads inside the enclosure (CT leads through a grommet); fuse each voltage sense lead. |
| SR-5 | Logger L floating on a mains-referenced circuit: the whole logger, power bank included, is at that potential. Keep the lid closed and don't touch it or plug in USB while connected. |
| SR-6 | Connect leads with the circuit de-energised where possible. |

## What the fan interposer would have needed (requirements scrub, D-11)

| Signal (interposer) | Type | Covered by |
|---|---|---|
| WH 15 V rail, sag and overnight bursts | DC, 0.05 V, BK referenced | L1 |
| YE / VSP from the unit's board | DC 0 to 7 V, BK referenced | L2 |
| YE / VSP as driven by the ESP32 | DC 0 to 7 V, BK referenced | L3 |
| Motor tach (BU), motor side and board side | 200 to 320 Hz pulses | F1, F2 |
| Remote In dry contact (5 V, 0.6 mA) | state | F1/F2 state or L4 |
| Unit mains supply and fan power | AC 120 V, watts | M1 |
| ~1 s motor-power drops | short dips | FR-L4 min/max |
| 160 V DC motor bus (RD) | high-voltage DC | not covered (D-12) |
| PWM duty, 1 kHz pickup on a floating tach, edge timing | waveform | oscilloscope |

All of the interposer's DC signals share BK as their reference, which is why
one floating Logger L with COM on BK covers them.

## Firmware

| File | What |
|---|---|
| `firmware/common.yaml` | ESP32-S3 base: Wi-Fi, API, OTA, web server |
| `firmware/mains-logger.yaml`, `mains-channel.yaml` | Logger M package (`pzemac`) |
| `firmware/lv-logger.yaml`, `lv-channel.yaml`, `freq-inputs.yaml` | Logger L package (`ina2xx_i2c` INA228, `pulse_counter`) |
| `firmware/example-mains.yaml`, `example-lv.yaml` | Device files to copy per logger |

## Change log

| Version | Date | Change |
|---------|------|--------|
| 0.1 | 2026-11-09 | First draft: D-1 to D-3, per-channel AMC3330 concept, open questions Q-1 to Q-3; D-4, D-5 settle Q-1 and Q-2. |
| 0.2 | 2026-11-09 | D-6 CH4 power channel with ADE7953 (settles Q-3), D-7 CT-only current. FR-13 to FR-16, CH4 hardware concept. D-8 CT for small loads (33 Ω burden). |
| 0.3 | 2026-11-09 | D-9 all through-hole: HCPL-7840 + SIP DC-DC on CH1-CH3. D-10 PZEM-004T module on CH4 (supersedes ADE7953 and D-8). FR-5, FR-13 to FR-16 and both hardware concepts rewritten. |
| 0.4 | 2026-11-09 | Requirements scrub against the interposer (D-11): FR-17 isolated frequency/state inputs F1, F2; FR-18 min/max per interval; table of interposer signals vs channels. Stock check: REM1-0505S + MCP1702 replace REM1-0509S + 78L05, Stackpole RNV replace Yageo HHV, Pomona 72930 jacks replace Staubli; LV top = 2 x 200 kΩ RNV14. |
| 0.5 | 2026-11-09 | D-12 split into Logger M (2 x PZEM-004T) and Logger L (4 x INA228 breakout, F1/F2, min/max, power bank). Combined HCPL-7840 design dropped (in git history, v0.4). Requirements renumbered FR-M, FR-L, FR-C, SR. |
