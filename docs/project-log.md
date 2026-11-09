# Project log

Newest last. One line per event; details live in the requirements doc.

- 2026-11-09: Project started (Scotty), from the Chiltrix interposer's WH-rail logger (ESP32-S3 + INA219 on a power bank). Goal: permanent, soldered, reusable multi-channel voltage logger with Home Assistant graphs.
- 2026-11-09: Decided: isolated per channel (D-1), AC and DC up to household voltage (D-2), own repo and generic firmware (D-3), 4 channels (D-4), HV and LV jack on every channel (D-5). INA226 idea dropped (36 V DC only); front end is now a fused HV divider into a TI AMC3330 per channel.
- 2026-11-09: Checked off-the-shelf options: isolated bench loggers (Hioki, Graphtec) cost $1.5k+ and don't feed HA; Bluetooth meters (Owon OW18E, UNI-T UT181A) are one channel each and need a BLE bridge. Kept the build.
- 2026-11-09: First draft of requirements, README and ESPHome package (not compiled yet).
- 2026-11-09: PR #1 merged. Scotty asked about the CS5460A; agreed on one power channel: CH4 = ADE7953 (chosen over CS5460A: in production, ESPHome driver, I2C) with clamp-on CT only (D-6, D-7; Scotty confirmed ADE7953). Requirements v0.2, firmware `power-channel.yaml`.
- 2026-11-09: CT sized for small loads (D-8): SCT-013-000 with 33 Ω burden, about 21 A RMS full scale.
- 2026-11-09: Scotty can't hand-solder SMD, so all through-hole (D-9, D-10): CH1-CH3 HCPL-7840 (DIP-8) + SIP isolated DC-DC; CH4 PZEM-004T v3.0 module (supersedes ADE7953 and the 33 Ω CT burden). Requirements v0.3.
- 2026-11-09: Digi-Key stock-check CSV (hardware/digikey-stock-check.csv) for Scotty's inventory check; PZEM-004T and enclosure bought elsewhere / later.
- 2026-11-09: Requirements scrub against the fan interposer (D-11, v0.4): its WH rail and two YE/VSP signals fit CH1-CH3 LV, but the tach comparison needed frequency inputs and the 1 s motor-power drops needed min/max. Added F1/F2 (LTV-827) and per-interval min/max. Waveform/PWM work stays with the scope.
- 2026-11-09: Scotty's Digi-Key check: Staubli jacks not carried, REM1-0509S and Yageo HHV non-stock (MOQ). Swapped to Pomona 72930-x jacks, REM1-0505S + MCP1702-5002 LDO, Stackpole RNV14/RNV12 HV resistors.
- 2026-11-09: Stock check 2: all lines in stock (about $162 with on-hand caps, $155 without; REM1-0505S $19 each is the biggest line). LV top changed to 2 x RNV14FAL200K (1/4 W parts share the mains fault); F1/F2 LED resistors use on-hand 10k 1/4 W (MFR-50 had no stock).
- 2026-11-09: Checked the KinCony MB (16 x INA226, 36 V DC only, $180): not a fit (no AC/mains, no isolation, no frequency inputs). No change.
- 2026-11-09: Split into two loggers (D-12, requirements v0.5): Logger M = ESP32 + 2 x PZEM-004T (mains V/I/W); Logger L = ESP32 + 4 x Adafruit INA228 (0-85 V DC + current, mean/min/max) + F1/F2 optos, power bank, shared COM. About $85-115 for both vs about $235 combined. Combined HCPL-7840 design dropped (git history). Firmware split into common/mains/lv packages; both example configs pass esphome config. New stock-check CSV.
- 2026-11-09: Stock check (split design): all 12 lines in stock, $123.58 at Digi-Key. Scotty's swaps kept: Adafruit 4401 cable, Keystone 4527 fuse holder, Bel 5HT 250-R fuses. Plus 2 x PZEM-004T (~$24, Amazon) and enclosures.
