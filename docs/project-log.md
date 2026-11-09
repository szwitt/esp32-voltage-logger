# Project log

Newest last. One line per event; details live in the requirements doc.

- 2026-11-09: Project started (Scotty), from the Chiltrix interposer's WH-rail logger (ESP32-S3 + INA219 on a power bank). Goal: permanent, soldered, reusable multi-channel voltage logger with Home Assistant graphs.
- 2026-11-09: Decided: isolated per channel (D-1), AC and DC up to household voltage (D-2), own repo and generic firmware (D-3), 4 channels (D-4), HV and LV jack on every channel (D-5). INA226 idea dropped (36 V DC only); front end is now a fused HV divider into a TI AMC3330 per channel.
- 2026-11-09: Checked off-the-shelf options: isolated bench loggers (Hioki, Graphtec) cost $1.5k+ and don't feed HA; Bluetooth meters (Owon OW18E, UNI-T UT181A) are one channel each and need a BLE bridge. Kept the build.
- 2026-11-09: First draft of requirements, README and ESPHome package (not compiled yet).
- 2026-11-09: PR #1 merged. Scotty asked about the CS5460A; agreed on one power channel: CH4 = ADE7953 (chosen over CS5460A: in production, ESPHome driver, I2C) with clamp-on CT only (D-6, D-7; Scotty confirmed ADE7953). Requirements v0.2, firmware `power-channel.yaml`.
