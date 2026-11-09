# ESP32 Voltage Logger: requirements

Version 0.4 (draft, 2026-11-09)

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
| D-6 | CH4 is a power channel (voltage, current, power); *chip choice superseded by D-10*; CH1-CH3 stay voltage-only. ADI ADE7953 chosen over the Cirrus CS5460A (still in production, ESPHome driver, I2C so a 2-wire isolator, two current inputs). | 2026-11-09 | Scotty (power channel), Claude (chip) |
| D-7 | CH4 current comes from a clamp-on split-core CT only (AC). No in-line shunt. | 2026-11-09 | Scotty |
| D-8 | *(Superseded by D-10.)* CH4 is sized for small loads: SCT-013-000 current-output CT (100 A : 50 mA) with a 33 Ω burden, about 21 A RMS full scale. For very small loads, loop the wire through the clamp N times and divide by N (set in calibration). | 2026-11-09 | Scotty (small loads), Claude (CT and burden) |
| D-9 | All through-hole: no surface-mount parts (Scotty has no SMD tooling). CH1-CH3 use a Broadcom HCPL-7840 isolation amplifier (DIP-8) instead of the AMC3330. | 2026-11-09 | Scotty |
| D-10 | CH4 uses a ready-made PZEM-004T v3.0 power meter module (100 A, split-core CT variant) instead of the ADE7953 (leadless package, no practical socket). Mounted on standoffs, screw terminals, isolated serial output. | 2026-11-09 | Scotty |
| D-11 | Requirements scrub against the fan interposer work: add 2 isolated frequency/state inputs (F1, F2) and min/max per report interval on CH1-CH3. Waveform, PWM duty and sub-millisecond work stay with the oscilloscope. | 2026-11-09 | Scotty (scrub), Claude (findings) |

## Notes

- DC current is not on the board; for low-voltage projects add an INA226 on the expansion header.
- CH4 (PZEM-004T) measures only 80 to 260 V AC; it reads nothing on low-voltage or DC circuits. Use CH1-CH3 for those.
- PZEM-004T current: 0.02 A starting current, 0.001 A resolution, so small loads down to about 20 mA are covered.

## Functional requirements

| ID | Requirement |
|----|-------------|
| FR-1 | 4 independent channels (D-4): CH1-CH3 voltage, CH4 power (D-6). |
| FR-2 | CH1-CH3 each report DC (mean) and AC (RMS, DC removed). True RMS (AC+DC) is derived in Home Assistant or firmware. |
| FR-3 | HV range: at least 300 V RMS / 425 V peak, AC (45 to 65 Hz) or DC. |
| FR-4 | LV range: about 42 V peak, resolution 0.05 V or better after averaging (D-5). |
| FR-5 | Each channel is isolated from every other channel and from USB/ESP32 (CH1-CH3: HCPL-7840 plus a reinforced-rated SIP DC-DC; CH4: the PZEM module's own optocoupled serial port). |
| FR-6 | Per channel, Home Assistant can select which range (HV or LV) is in use; the choice survives reboots. |
| FR-7 | Per channel calibration (gain and zero offset) set in YAML substitutions, no rebuild of hardware. |
| FR-8 | Channel names, update rates and calibration live in a small per-use device YAML; the shared package does not change between projects. |
| FR-9 | Report rate configurable, default 1 s. |
| FR-10 | Powered from a normal USB-C charger; no power bank needed because the inputs are isolated. |
| FR-11 | Expansion header: I2C (3.3 V) and one 1-Wire pin, for extra sensors (temperature, INA219/INA226 for low-voltage current). |
| FR-12 | Works without Home Assistant too: ESPHome web server shows live readings. |
| FR-13 | CH4 reports AC voltage RMS, frequency, CT current RMS, active power, energy and power factor (D-6, D-7, D-10). AC mains only, 80 to 260 V. |
| FR-14 | CH4 voltage: one HV jack + COM wired to the PZEM's L/N terminals, fused. Resolution 0.1 V. |
| FR-15 | CH4 current: the PZEM's own split-core 100 A CT, its leads on the PZEM's screw terminals (D-10). |
| FR-16 | CH4 calibration: optional voltage, current and power multipliers in YAML substitutions (module is factory calibrated). |
| FR-17 | Two isolated digital inputs F1, F2 (IN + COM jack each), 3 to 30 V signals, up to 5 kHz. Each reports frequency (with a per-input scale, for example rpm = Hz x 60 / 12 for a 12-pulse tach) and on/off state. Optocoupler (LTV-827 dual) into ESP32 pulse counters GPIO10/GPIO11 (D-11). |
| FR-18 | CH1-CH3 also report the minimum and maximum over each report interval, so short dips (about 1 s, like the interposer's motor-power drops) show up even at a 1 s or slower report rate (D-11). |

## Hardware concept: voltage channels CH1-CH3

```
HV jack ──[fuse/flameproof R]──[4 x 1 MΩ HV string]──┐
LV jack ──[2 x 200 kΩ HV, in series]─────────────┤
                                                     ├── HCPL-7840 VIN+ ──║── VOUT+/VOUT- ── op-amp (diff -> single-ended, 1.65 V mid)
COM jack ──────────────────────[1.69 kΩ]─────────────┘    HCPL-7840 VIN-  ║               └── ESP32-S3 ADC1 pin
                                                          VDD1 5 V        ║  VDD2 5 V (USB side)
   isolated 5 V: REM1-0505S SIP DC-DC (reinforced, 5.2 kVDC) -> MCP1702-5002 LDO -> VDD1
```

- **Isolation amplifier:** Broadcom HCPL-7840-000E, through-hole DIP-8 (D-9).
  ±200 mV linear input (±300 mV clip), fixed gain 8, AC and DC. Gain tolerance
  about ±5%, so each channel is calibrated (FR-7).
- **Input-side supply:** one RECOM REM1-0505S (SIP7, 1 W, 5.2 kVDC reinforced)
  per channel, then an MCP1702-5002 low-dropout regulator (TO-92) so VDD1
  stays inside 4.5 to 5.5 V whatever the module's light-load output.
- **Jacks:** Pomona 72930-x panel jacks (IEC 61010, for sheathed plugs) on
  the enclosure lid, wired to the board with 600 V rated wire and ring lugs.
- **Scaling (nominal):** HV 425 V peak and LV 42 V peak both map to about
  ±0.18 V at the HCPL input, ±1.44 V differential at its output, and (op-amp
  gain 0.75) about ±1.08 V at the ADC around 1.65 V. Overall gain stays about
  393 V (HV) and 40 V (LV, 400 kΩ top) per ADC volt, so the firmware defaults hold.
- **Fault case:** mains on the LV jack: the two 200 kΩ (1600 V rated) share
  about 0.3 W at 240 V (0.15 W each, 1/4 W parts) and the HCPL input sees about ±1.5 V peak, inside its
  absolute rating (-2 V to VDD1 + 0.5 V; to confirm on the datasheet), so the
  reading clips but nothing is damaged.
- **Op-amp:** MCP6002 (DIP-8, dual, 3.3 V rail to rail): one dual per two channels.
- **ADC:** ESP32-S3 ADC1 (GPIO1, GPIO2, GPIO5, GPIO6; GPIO3 is a strapping pin), calibrated per channel. A dedicated
  ADC is a possible later upgrade if the ESP32 ADC proves too noisy.
- **Board:** small 2-layer PCB (OSH Park), ESP32-S3-DevKitC-1 on headers.

## Hardware concept: frequency inputs F1, F2

```
F1 IN jack ──[10 kΩ 1/2 W]──┬── LTV-827 LED A ──║── collector ── GPIO10 (pull-up 10 kΩ to 3.3 V)
F1 COM jack ────────────────┴── (reverse diode across LED)
F2: same, LTV-827 second half -> GPIO11
```

- About 0.3 mA at 5 V and 3 mA at 30 V into the LED; the LTV-827's
  phototransistor is slow (tens of µs), fine to about 5 kHz.
- What the interposer needed it for: the motor's tach (BU, 12 pulses per rev,
  about 200 to 320 Hz) on the motor side and on the board side at once, to see
  the synthetic-tach handback creep, and the Remote In dry contact state.

## Hardware concept: power channel CH4

```
HV jack ──[fuse]──┐                          ┌── PZEM-004T v3.0 (100 A) L/N screw terminals
COM jack ─────────┘                          │   CT (split-core) on its CT screw terminals
                                             └── TTL side (5 V, optocoupled): TX/RX -> ESP32 UART GPIO17/18
```

- The PZEM-004T is a ready-made module (D-10): mount it on standoffs inside
  the enclosure, no fine soldering. ESPHome driver `pzemac` (Modbus over UART,
  9600 baud).
- Its TTL side is powered from 5 V; its TX/RX optocouplers work with the
  ESP32's 3.3 V logic (some modules need a pull-up change; check on bring-up).
- The CT clamp's leads go to the module's screw terminals; the CT is
  isolated from the conductor, but its leads connect to the module's mains
  side, so route them inside the enclosure through a grommet, not a jack.

## Safety requirements

| ID | Requirement |
|----|-------------|
| SR-1 | Aim at CAT II 300 V practice. This is a home-built, uncertified instrument; use it with that in mind. |
| SR-2 | Creepage across each channel's barrier and between channels at least 6.4 mm, with board slots under each HCPL-7840 and SIP DC-DC, and around the PZEM module's mains terminals. |
| SR-3 | Shrouded 4 mm banana jacks (Pomona 72930-x, IEC 61010) and CAT II (or better) rated sheathed leads only. Jack-to-board wires 600 V rated. |
| SR-4 | Every HI input has a fuse or flameproof series resistor; HV divider split over several resistors so none sees more than its rated voltage. |
| SR-5 | Closed plastic enclosure; no exposed metal connected to any input. USB and expansion header are on the safe (ESP32) side only. |
| SR-6 | Connect leads with the circuit de-energised where possible. |

## What the fan interposer would have needed (requirements scrub, D-11)

| Signal (interposer) | Type | Covered by |
|---|---|---|
| WH 15 V rail, sag and overnight bursts | DC, 0.05 V, BK (mains) referenced | CH1 LV |
| YE / VSP from the unit's board | DC 0 to 7 V, BK referenced | CH2 LV |
| YE / VSP as driven by the ESP32 | DC 0 to 7 V, BK referenced | CH3 LV |
| Motor tach (BU), motor side and board side | 200 to 320 Hz pulses | F1, F2 (new) |
| Remote In dry contact (5 V, 0.6 mA) | state | F1/F2 state, or a CH LV |
| Unit mains supply and fan power | AC 120 V, watts | CH4 |
| 160 V DC motor bus (RD) | DC | CH HV range |
| ~1 s motor-power drops | short dips | FR-18 min/max |
| PWM duty, 1 kHz pickup on a floating tach, edge timing | waveform | oscilloscope (out of scope) |

## Firmware

ESPHome package `firmware/voltage-logger.yaml`, included from a per-use device
file (see `firmware/example-device.yaml`). First cut uses ESPHome's `adc` plus
`ct_clamp` components for CH1-CH3 and `pzemac` for CH4
(`firmware/power-channel.yaml`); a small custom component that computes DC mean, AC RMS,
min and max in one sampling pass is planned (FR-18 min/max is done in YAML for now). F1/F2 use `pulse_counter` (`firmware/freq-inputs.yaml`).

## Change log

| Version | Date | Change |
|---------|------|--------|
| 0.1 | 2026-11-09 | First draft: D-1 to D-3, per-channel AMC3330 concept, open questions Q-1 to Q-3; D-4, D-5 settle Q-1 and Q-2. |
| 0.2 | 2026-11-09 | D-6 CH4 power channel with ADE7953 (settles Q-3), D-7 CT-only current. FR-13 to FR-16, CH4 hardware concept. D-8 CT for small loads (33 Ω burden). |
| 0.3 | 2026-11-09 | D-9 all through-hole: HCPL-7840 + SIP DC-DC on CH1-CH3. D-10 PZEM-004T module on CH4 (supersedes ADE7953 and D-8). FR-5, FR-13 to FR-16 and both hardware concepts rewritten. |
| 0.4 | 2026-11-09 | Requirements scrub against the interposer (D-11): FR-17 isolated frequency/state inputs F1, F2; FR-18 min/max per interval; table of interposer signals vs channels. Stock check: REM1-0505S + MCP1702 replace REM1-0509S + 78L05, Stackpole RNV replace Yageo HHV, Pomona 72930 jacks replace Staubli; LV top = 2 x 200 kΩ RNV14. |
