# AHRS-ESP32

Attitude and Heading Reference System on an ESP32-S3. Raw IMU + magnetometer in, quaternion attitude with gyro-bias estimation out. Acquisition and estimation run in bounded time on one core; telemetry (UART or UDP) runs on the other; a Processing GUI on the PC displays and commands it.

Portfolio project aimed at defence/avionics embedded roles, built to be reused in a future UAV. Its role there is undecided (standalone module on a bus, or library inside an autopilot), so both must stay possible.

Current state, pin map, measurements and open decisions are in the session context file, imported here:

@docs/SESSION_CONTEXT.md

## Non-negotiable rules

1. **Zero heap in our code.** No `new`/`delete`/`malloc`/`free`, no allocating std types (`vector`, `string`, `function`, `map`, smart pointers, iostreams). FreeRTOS objects only through the `*CreateStatic*` APIs. ESP-IDF internals (Wi-Fi, lwIP, drivers) may allocate during init; the real-time core never calls anything that allocates after init. Enforced by `tools/check_no_heap.sh`.
2. **No third-party code.** Allowed: ESP-IDF (vendor SDK, includes FreeRTOS), non-allocating C++17 standard headers, Processing core with `processing.serial` and `java.net`. Not allowed: Arduino core, Eigen, sensor/filter/protocol libraries, test frameworks, IDF managed components. Drivers, maths, filters, protocol and test harness are ours.
3. **C++17, float only.** `-Wall -Wextra -Werror -Wconversion -Wshadow -Wdouble-promotion -fno-exceptions -fno-rtti`. No `double` (the S3 FPU is single precision): `float` and `f`-suffixed literals. No recursion. Every loop has a static bound. If IDF headers trip the strict flags, isolate that include behind a thin wrapper; never relax flags globally.
4. **Bounded time in the RT path.** Nothing there blocks without a timeout, logs (`ESP_LOGx`, `printf`), takes a lock shared with core 0, or touches flash.
5. **Portable core.** Everything under `libs/` is pure C++17 with no ESP-IDF or FreeRTOS includes, and builds and unit-tests on the host.
6. **Never guess hardware facts.** Register addresses, bit fields, timings and scale factors come from the datasheets in `docs/datasheets/` (user-supplied, git-ignored). If it is not there, ask.
7. **Check ESP-IDF APIs against the installed, pinned IDF headers**, not from memory.

If a rule cannot be met, stop and ask. Do not work around it silently.

## Conventions

- Frames: body FRD (x forward, y right, z down), navigation NED. At rest and level the accelerometer reads specific force ≈ (0, 0, −9.81) m/s².
- Quaternion: Hamilton, scalar first `[w x y z]`, unit norm, `q_nb` rotates body vectors into nav. Euler angles for display only.
- SI units, with the unit in the identifier: `_rad`, `_rps`, `_mps2`, `_s`, `_us`.
- Namespace `ahrs::`. Types `PascalCase`, functions and variables `snake_case`, constants `kPascalCase`.
- Fixed-width integers, `enum class`, `constexpr`, `static_assert`. Errors are `[[nodiscard]] enum class Status` return values.
- Virtual dispatch only at component boundaries (sensor, transport, estimator), never inside numeric kernels.
- English for code, comments, docs and commit messages.
- MISRA C++:2023 is guidance. Do not claim compliance anywhere.

## Architecture

**Core 1 (APP), real time**
- DRDY GPIO ISR, in IRAM: capture timestamp, notify `rt_task`. No SPI, no float.
- `rt_task`, highest application priority, pinned: SPI burst read → scale and calibrate → estimator step → push result into the SPSC ring.

**Core 0 (PRO), communications**
- Wi-Fi and lwIP live here (IDF default).
- `comms_task`: pops the ring, frames, sends on the active transport, receives commands, owns NVS. NVS writes only outside RUN, because flash operations can stall both cores.

**Coupling between cores**
- One wait-free SPSC ring of ours (`std::atomic` indices). The producer never blocks; overflow drops the sample and increments a counter.
- Commands travel back through a second mailbox, applied by `rt_task` at a cycle boundary.

**Supervision**
- IMU deadline: `rt_task` waits for DRDY with a timeout of 1.5 periods. A miss increments a counter and moves the state to DEGRADED.
- Task watchdog: fed by `rt_task` only after a complete, in-budget cycle with fresh IMU data. Sustained DRDY loss or a hang means it is not fed, and the chip resets. Reset reason is reported at boot.
- States: BOOT → INIT → ALIGN → RUN ⇄ DEGRADED → FAULT. Attitude is never flagged valid outside RUN.

**Estimator**
- Multiplicative error-state Kalman filter (MEKF), 6 error states: attitude error (3) and gyro bias (3); the nominal quaternion is propagated with bias-corrected gyro.
- Accelerometer corrects roll and pitch; magnetometer corrects yaw only.
- Sequential scalar updates: no matrix inverse, fixed execution time.
- Mahony complementary filter as the baseline behind the same interface.

**Initial targets** (replace with measurements as they arrive): IMU 1 kHz, magnetometer 100 Hz, attitude telemetry 100 Hz, RT cycle worst case ≤ 400 µs, zero missed samples over 1 h with Wi-Fi streaming.

**Hardware**: ESP32-S3 dev board; IMU ICM-42688-P on SPI with INT1 as DRDY (alternative ICM-45686); magnetometer to be chosen. Take pins from the module datasheet, avoiding strapping, USB and flash/PSRAM pins.

## Repository layout

```
CLAUDE.md
docs/SESSION_CONTEXT.md     living hand-off between sessions
docs/design/                one design note per milestone, protocol spec, MEKF derivation
docs/results/               histograms, logs, plots backing every published number
docs/datasheets/            user-supplied, git-ignored
libs/linalg/                fixed-size matrix, vector, quaternion (header-only)
libs/ahrs_core/             estimators, calibration models, SPSC ring
libs/protocol/              framing, CRC, message serialisation
firmware/                   ESP-IDF project: main/, components/{bsp,drivers,rt,comms}
host/                       CMake build of libs, tests/, tools/replay
gui/ahrs_viewer/            Processing sketch
tools/                      check_no_heap.sh and other scripts
```

## Commands

Created in M0; update this section if they change.

```
cmake -S host -B build/host && cmake --build build/host
ctest --test-dir build/host --output-on-failure
idf.py -C firmware build
idf.py -C firmware -p <PORT> flash monitor
tools/check_no_heap.sh
```

## Session protocol

One milestone per session.

**Start**
1. Find the active milestone in the session context and read the files it lists.
2. Present a plan: files to create or change, public interfaces, test strategy, open questions. Wait for approval before writing code.

**During**
- Stay inside the milestone. Out-of-scope findings go to the Backlog section of the session context.
- Tests are written with the code, not after.
- Small commits, `M<n>: <what>`. Never push.

**End (definition of done)**
- Host build and tests pass with ASan and UBSan; firmware builds with zero warnings; `tools/check_no_heap.sh` passes.
- Acceptance criteria met. For checks that need the board, give the user exact steps and the expected output, then record what they report.
- `docs/design/M<n>-<slug>.md` written: what was built, why, alternatives rejected, measured numbers. The owner must be able to defend every decision in an interview from this note.
- Session context updated: status, decisions with reasons, measurements, deviations from this file, what the next milestone needs.
- Milestone status updated below.

## Milestones

Status: `[ ]` todo, `[~]` in progress, `[x]` done.

### M0 — Toolchain and scaffold `[ ]`
Repository layout; ESP-IDF project for `esp32s3` on the latest stable IDF (pin and record the version); compiler flags; static-allocation FreeRTOS config; host CMake with sanitizers; own minimal test harness; `tools/check_no_heap.sh` (inspects our archives for undefined `malloc`/`free`/`operator new`/`operator delete` symbols); CI running host tests and a firmware build; `.gitignore`; README stub.
**Done when:** firmware with two static tasks pinned to different cores runs; a dummy host test passes; CI is green; the heap check passes, and fails when a `new` is added on purpose.

### M1 — `linalg` `[ ]`
Header-only `Matrix<Rows, Cols>` (float, `std::array` storage, dimensions checked at compile time), `Vec3`, `Quat`. Add, subtract, multiply, transpose, scale, identity, skew, symmetrise; quaternion product, conjugate, normalise, rotate; small-angle vector → quaternion; quaternion ↔ DCM; quaternion → Euler. `constexpr` where practical. No general inverse (not needed).
**Done when:** tests cover every operation, algebraic identities and edge cases (near-zero normalisation, pitch at ±90°); builds warning-free on host and target.

### M2 — IMU driver `[ ]`
Own SPI driver written from the datasheet: bus init, identity check, reset, 1 kHz output rate, full-scale ranges, on-chip filter config, data-ready interrupt enable, burst read of accel + gyro + temperature, raw → SI. Fixed sensor-to-body rotation in `bsp`. Narrow `ImuSensor` interface so the part can be swapped. Polled bring-up only; logging is allowed here. Pin map recorded in the session context.
**Done when:** at rest the user sees |a| ≈ 9.81 and gyro ≈ 0; each axis has the correct sign for FRD; SPI failures return a status; no heap, no unbounded wait.

### M3 — Real-time acquisition pipeline `[ ]`
DRDY ISR → notification → `rt_task` on core 1; microsecond timestamps; deadline supervision; watchdog policy; state machine; fixed-bucket histograms for sample period, ISR-to-task latency and cycle execution time; SPSC ring (in `libs/`, host-tested including a two-thread stress test) feeding `comms_task` on core 0, which for now prints statistics at 1 Hz.
**Done when:** a 10-minute run shows zero missed samples and the histograms are saved in `docs/results/`; disconnecting DRDY gives DEGRADED, then a watchdog reset with the right reset reason; a busy task loading core 0 leaves the core-1 histograms unchanged.

### M4 — Telemetry protocol and UART `[ ]`
`libs/protocol`: versioned binary framing (sync, id, length, sequence, payload, CRC-16), explicit little-endian field serialisation with no struct punning, streaming parser that resynchronises after garbage or truncation. Messages RAW_IMU, ATTITUDE, HEALTH, CMD, ACK; per-message rates set by command. `Transport` interface with a UART implementation. Command path PC → `comms_task` → `rt_task` mailbox. Spec in `docs/design/protocol.md`; the GUI is written from that spec.
**Done when:** host tests pass (round trip, corruption, resync, fixed-seed fuzz); 10 minutes on target with zero CRC errors and sequence gaps reported; RT histograms match M3.

### M5 — Baseline estimator `[ ]`
`libs/ahrs_core`: `Estimator` interface (align, step with gyro/accel/dt, optional mag; outputs quaternion, bias, validity flags). Mahony filter with integral bias term. Initial alignment from the accelerometer. Synthetic trajectory generator for host tests with known truth and configurable noise and bias.
**Done when:** host tests show convergence from a wrong initial attitude and bias tracking within stated tolerances; it runs in `rt_task` with ATTITUDE streamed; execution time is recorded.

### M6 — Processing GUI v1 `[ ]`
`gui/ahrs_viewer` (Java mode, P3D): port selection, protocol decoder, 3D body driven by the quaternion, strip charts (raw sensors, Euler, bias), health panel (state, jitter, worst-case time, drops, CRC errors), raw-log recording to a binary file using the same framing.
**Done when:** the 3D model follows the board with the correct sense on all three axes; a recorded log decodes with the host parser.

### M7 — Magnetometer `[ ]`
Own driver from the datasheet behind a `MagSensor` interface; sampling near 100 Hz with bounded bus transactions; RAW_MAG message; GUI plot and 3D scatter of samples; Mahony extended with heading correction.
**Done when:** RT histograms stay within budget; heading responds correctly to rotation about the vertical; disconnecting the magnetometer is flagged and the system continues with heading marked invalid.

### M8 — Calibration `[ ]`
Gyro bias at start-up with stillness detection; accelerometer six-position offset and scale; magnetometer hard and soft iron fitted on the PC with own least-squares code; parameters uploaded by command, stored in NVS with version and CRC, applied in `rt_task`. Guided workflow in the GUI.
**Done when:** |a| at rest is within tolerance in six orientations; mag samples lie on a sphere with the residual reported; parameters survive a reboot.

### M9 — MEKF I: propagation and gravity update `[ ]`
Design note first (`docs/design/mekf.md`): states, error dynamics, discrete transition and process noise from gyro noise density and bias random walk, accelerometer measurement model, reset step. Then the implementation behind `Estimator`: covariance propagation, sequential scalar updates, symmetry preservation, accelerometer gating (norm and innovation) against dynamic acceleration.
**Done when:** on synthetic data attitude and bias converge, innovation statistics are consistent with the predicted covariance, a one-hour simulated run does not diverge, and the covariance stays symmetric and positive.

### M10 — MEKF II: heading update and target integration `[ ]`
Magnetometer update restricted to yaw, with innovation gating; alignment including heading; estimator selectable at run time by command; covariance diagonal and gate flags in telemetry and GUI; worst-case time measured on target.
**Done when:** the RT cycle is within budget with the MEKF; a magnet near the board trips the heading gate while roll and pitch stay put; bias at rest agrees with the start-up estimate from M8.

### M11 — Replay and noise characterisation `[ ]`
`host/tools/replay`: runs recorded raw logs through the same `ahrs_core` code and writes CSV; Mahony versus MEKF comparison. Allan deviation tool on a long static log → noise density and bias instability → process and measurement noise; tuning documented.
**Done when:** replay reproduces the on-target quaternion within a documented tolerance; the tuned parameters trace to measured Allan curves in `docs/results/`.

### M12 — Wi-Fi transport `[ ]`
UDP transport; soft-AP by default, station mode optional with credentials sent by command and stored in NVS (never in the repository). Both transports always accept commands; telemetry goes to whichever last requested the stream, which is how the GUI selects serial or Wi-Fi. Link-loss handling. GUI transport selector using `java.net.DatagramSocket`.
**Done when:** switching transport from the GUI works in both directions without a reboot; a one-hour Wi-Fi run has zero missed IMU samples and histograms equivalent to the UART-only run, both saved.

### M13 — Fault injection and robustness `[ ]`
Command- or build-flag-driven injection: DRDY loss, SPI errors, stuck or saturated data, NaN guards, ring overflow, core-0 stall, reset. For each: detection, state transition, telemetry code, recovery. Fault table in the docs.
**Done when:** every row of the fault table is demonstrated and recorded; no fault leaves ATTITUDE flagged valid while wrong.

### M14 — Release `[ ]`
README with architecture diagram, design rationale, build and run instructions, wiring, results (jitter histograms, worst-case time, static drift, bias convergence, Mahony versus MEKF) and limitations stated plainly. Tag v1.0.
**Done when:** a fresh clone builds by following the README; every number in the README traces to a file in `docs/results/`.

## Out of scope for v1

GPS/barometer and the 15-state INS extension, MAVLink or CAN output, redundant IMUs, temperature compensation, control loops. Do not build them, and do not block them: keep the `Estimator`, `Transport` and sensor interfaces narrow.
