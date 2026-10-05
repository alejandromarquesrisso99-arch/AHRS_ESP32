# Session context

Living hand-off between Claude Code sessions. `CLAUDE.md` holds the rules and the plan; this file holds the current state. Update it at the end of every session. Keep it short: detail belongs in `docs/design/`.

## Status

- Active milestone: **M0 — Toolchain and scaffold**
- Last session: none yet
- ESP-IDF version: not pinned yet (M0)

## Decisions

| Date | Decision | Reason |
| --- | --- | --- |
| 2026-10-05 | MCU: ESP32-S3 | Dual core, single-precision FPU, owner's choice |
| 2026-10-05 | ESP-IDF only, no Arduino core | Static FreeRTOS objects, core pinning and IRAM control are needed |
| 2026-10-05 | Estimator: 6-state MEKF, Mahony as baseline | Estimates gyro bias, fixed execution time, extends to a 15-state INS later |
| 2026-10-05 | Estimation on core 1, comms on core 0 | Wi-Fi and lwIP run on core 0 by default |
| 2026-10-05 | Core logic in portable `libs/` | Role in the future UAV is undecided; host testing and replay |
| 2026-10-05 | Own test harness, no test framework | No-third-party rule |
| 2026-10-05 | English for code and docs | Public repository for international recruiters |

## Open decisions

| Needed by | Decision |
| --- | --- |
| M0 | Exact ESP32-S3 board and module (flash/PSRAM variant) |
| M2 | IMU part: ICM-42688-P (default) or ICM-45686; breakout bought and datasheet in `docs/datasheets/` |
| M4 | Serial path: on-board USB-UART bridge or native USB |
| M7 | Magnetometer part (candidates: MMC5983MA, LIS3MDL); datasheet in `docs/datasheets/` |

## Hardware

- Board: to be filled in M0
- Pin map: to be filled in M2
- Sensor-to-body rotation: to be filled in M2

## Measurements

None yet. Every number recorded here must point to a file in `docs/results/`.

## Deviations from CLAUDE.md

None.

## Notes for the next session

- Nothing exists yet; M0 starts from an empty repository.

## Backlog

Out-of-scope findings noted during sessions.
