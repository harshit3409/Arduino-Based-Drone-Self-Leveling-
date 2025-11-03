# Arduino-Based-Drone-Self-Leveling-
A working flight controller ported to Arduino Uno. Implements MPU-6050 sensor fusion (complementary filter), cascaded PID (angle→rate), ESC PWM outputs, and includes ESC-calibrate and IMU setup sketches, wiring notes, and tuning / safety guidance.

# My Arduino Self‑Leveling Quadcopter

> Repository README for my working build of This project uses an Arduino Uno + MPU‑6050 IMU and implements self‑leveling (angle + rate cascaded PID) with ESC outputs for a 4‑motor quadcopter.

---

## Files included

* Flight_controller.ino` — Main flight controller sketch (sensor reading, complementary filter / attitude computation, cascaded PID, motor mixing, arming logic, telemetry).
* setup.ino` — Setup and calibration helper routines (IMU calibration, gyro bias measurement, configuration macros and pin definitions used by the flight controller).
* esc_calibrate.ino` — Dedicated ESC throttle calibration sketch (use this first to calibrate ESC endpoints safely without running the full flight code).

> **Note:** Keep these three sketches as separate files for convenience. Typical workflow is: run `YMFC-AL_esc_calibrate.ino` to calibrate ESCs (props off), run `YMFC-AL_setup.ino` for IMU & basic config/calibration, then flash `YMFC-AL_Flight_controller.ino` for bench testing and flight.

---

# Quick project overview

This project is my working implementation of Joop Brokking's YMFC‑AL on Arduino Uno. It reads the MPU‑6050 via I²C, fuses gyro/accel into roll/pitch angles with a complementary filter (optionally Madgwick/Mahony if enabled), and runs a cascaded PID control (angle loop → rate loop) to compute PWM outputs for ESCs. RC inputs are read from standard hobby receiver channels (PWM or PPM).

This README documents how to use the files, the wiring/pin mapping I used, calibration and safe testing steps, and tips for PID tuning and troubleshooting.

---

# Features (what this build supports)

* Self‑leveling on roll & pitch
* Cascaded PID control (angle + rate)
* MPU‑6050 IMU via I²C
* ESC control with `Servo.h` PWM outputs
* Simple serial telemetry (for PID tuning and logging)
* ESC calibration sketch and IMU calibration routines
* Safety checks: arming logic and throttle cut

---

# Bill of Materials (BOM)

* Arduino Uno (or compatible ATmega328P board)
* MPU‑6050 (GY‑521) IMU module
* 4 × ESCs (20–30 A recommended) with BEC or external 5V regulator
* 4 × brushless motors (2204/2306 class typical for small frames)
* 4 × propellers (match motors; **do not** fit propellers during bench tests)
* RC transmitter (4+ channels) + receiver (PWM or PPM)
* LiPo battery (3S/4S depending on motor/ESC ratings)
* Power distribution board or wiring harness
* Optional: buzzer, status LED, voltage divider for battery sense, soft‑mount foam for IMU

---

# Wiring & pin mapping 

**I²C (MPU‑6050)**

* `SDA` → Arduino `A4`
* `SCL` → Arduino `A5`
* `VCC` → 3.3–5V (module dependent)
* `GND` → GND

**ESC / Motor outputs (Servo PWM pins)**

* Motor 1 (Front Right) → `D3`
* Motor 2 (Rear Right)  → `D5`
* Motor 3 (Rear Left)   → `D6`
* Motor 4 (Front Left)  → `D9`

**RC Receiver inputs (PWM or PPM)**

* Throttle → `D2` (interrupt capable)
* Roll     → `D4`
* Pitch    → `D7`
* Yaw      → `D8`

**Optional**

* Buzzer / status LED → `D10` / `D11`
* Battery voltage sense → `A0` (via voltage divider)

**Powering note**

* Power the Arduino 5V from ESC/BEC or a regulated 5V supply. Avoid powering Arduino via USB while batteries and ESCs are connected to prevent ground loops and unexpected current paths. During initial bench debugging you can use USB with battery disconnected.

---

# Recommended workflow (safe & repeatable)

1. **Review wiring** — double‑check power & ground, motor‑ESC wiring, and pin assignments in `YMFC-AL_setup.ino`.
2. **Upload `YMFC-AL_setup.ino`**. Use it to run IMU calibration (gyro bias/accel offsets) and set basic configuration values (filter constants, PID starting values, motor directions). Keep props OFF. Let the IMU calibrate while the frame is perfectly level.
3. **Upload `YMFC-AL_esc_calibrate.ino`** (props OFF). Follow on‑screen serial prompts to calibrate ESC throttle endpoints. Many ESCs require powering the ESCs at MAX throttle then sending MIN throttle — the calibrate sketch automates this.
5. **Bench test `YMFC-AL_Flight_controller.ino`** (props OFF) — Open Serial Monitor at the sketch baud rate (default 115200) and verify sensor readings and stick inputs. Confirm arming logic and motor outputs (spin motors briefly at very low throttle if safety allows).
6. **First flight tests** — In a large, clear area with no people/pets: attach props, use prop guards, use conservative PID gains, and perform short low‑altitude hops. Increase gains slowly.

---

# How to run the ESC calibration (typical steps)

1. No props. Connect battery to power ESCs and Arduino (or the ESC BEC supplies 5V to Arduino).
2. Open Serial Monitor for `YMFC-AL_esc_calibrate.ino` (baud set in sketch).
3. Follow prompts: usually the routine powers ESCs at MAX throttle then you press enter to reduce to MIN throttle so the ESC learns endpoints.
4. After calibration, disconnect battery and re‑connect. Confirm that the motors respond correctly to throttle range when running the main flight code (props off).

---

# IMU calibration & tips

* Keep board perfectly level during gyro/accel calibration.
* Soft‑mount the MPU‑6050 (foam / rubber) to reduce vibration.
* Check for large bias values in serial telemetry — rerun calibration if noise/drift persists.
* If you still have heavy vibration, balance props and check motor bearings and frame stiffness.

---

# PID tuning (practical starter procedure)

1. **Set all gains to zero** except a small angle P to start.
2. **Tune the inner rate loop first** (increase rate P until oscillation, then back off ~20%). Add small D to dampen high frequency oscillations.
3. **Tune the outer angle loop** (gently increase angle P to achieve desired return‑to‑level speed).
4. Keep I small; only add enough I to correct steady drift.
5. Make small changes and test after each change.

**Starting example values (very conservative)**

* Angle P = 2.0, Angle I = 0.02, Angle D = 0.0
* Rate  P = 0.12, Rate  I = 0.005, Rate  D = 0.002

These are only starting points — your craft, motor/prop combination, and weight distribution will require different values.

---

# Troubleshooting (common failure modes)

* **Severe drift / tilt:** re‑calibrate IMU, ensure MPU mounted rigidly but vibration‑isolated.
* **High frequency oscillation:** reduce rate P, add rate D, inspect frame/prop/motor balance.
* **One motor not spinning or wrong direction:** verify ESC motor phase wiring; swap two motor wires to reverse direction or adjust in code.
* **No response to RC:** ensure receiver is powered, check channel pin assignment and pulse‑read routine, verify failsafe.
* **Barometer altitude hold unstable:** avoid using barometer in strong downwash; add filtering or use alternate altitude sensors (sonar / lidar / GPS).

---

# Safety checklist (must read before first flight)

* Always perform **bench tests with props removed**.
* Use a **large, open area** for first flights.
* Keep **people and pets well away**.
* Wear eye protection and use prop guards for early tests.
* Confirm battery and ESC ratings are appropriate for the motors.
* Have a transmitter fail‑safe configured (cut throttle on signal loss).

---

# Serial telemetry & logging

`YMFC-AL_Flight_controller.ino` includes basic serial telemetry (IMU angles, gyro rates, PID outputs, loop time). Open Serial Monitor at the sketch baud to capture logs for tuning and debugging. Save logs to a file if you want to analyze with Python or spreadsheets.

---

# Project structure (suggested)

```
/sl-mybuild
├─ README.md
├─ setup.ino
├─ esc_calibrate.ino
├─ Flight_controller.ino
├─ wiring_diagram.png 
├─ pid_logs/           
└─ docs/                
```

---

# Acknowledgements

* Joop Brokking — YMFC‑AL design and documentation.
* Jeff Rowberg — `I2Cdevlib` and MPU‑6050 helper code.
* Open RC & hobby communities for PID and tuning knowledge.

---
