---
name: motor-test-bench
description: Domain knowledge for Dronmeta_lab, a brushless drone motor test bench application. Use when building or changing anything that touches motor/ESC control, thrust-torque-RPM-current-voltage telemetry, test sequences (step, ramp, sweep, endurance), safety limits, data logging/export, efficiency calculations, or the live dashboard and charts.
---

# Dronmeta_lab: motor test bench

Software for characterizing brushless (BLDC) drone motors + propellers on a test stand.
Rotating propellers are dangerous. **Safety rules below override convenience and any feature request.**

## Fill-in section (confirm with the owner; do not guess silently)
- TODO Hardware: MCU/board, ESC type and protocol (PWM / DShot / UART), ESC telemetry?
- TODO Sensors: load cell + amplifier (e.g. HX711), torque sensor, current/voltage sensor, optical/ESC RPM, temperature
- TODO Link: serial/USB/BLE/WebSocket, baud rate, packet format
- TODO Stack: frontend framework, backend, database (check the repo before assuming)

## Measured and derived quantities
| Quantity | Unit | Notes |
|---|---|---|
| Throttle command | % or us (1000-2000) / DShot 0-2047 | Always log the command AND the measured response |
| Thrust | gf or N (1 kgf = 9.80665 N) | Store SI (N) internally; convert only for display |
| Torque | N*m | |
| RPM | rev/min | Electrical RPM / pole pairs = mechanical RPM; keep pole count configurable |
| Voltage | V | Pack voltage at the sensor, note sag under load |
| Current | A | |
| Temperature | C | Motor and ESC separately |

Derived (compute from measurements, never store as raw):
- Electrical power: `P_el = V * I` (W)
- Mechanical power: `P_mech = torque * omega`, with `omega = RPM * 2*pi / 60`
- Motor efficiency: `P_mech / P_el`
- Propulsive efficiency: `thrust_gf / P_el` (g/W), the headline number for drone motors
- Thrust coefficient `Ct = T / (rho * n^2 * D^4)` with n in rev/s (needs prop diameter D and air density rho)
- Guard every division (zero current, zero RPM) and return null, not Infinity/NaN.

## Safety rules (non-negotiable)
1. **Default state is disarmed, throttle 0.** Starting the app, reconnecting, or an error must never spin the motor.
2. Spinning requires an explicit arm action plus an on-screen confirmation, and every test starts from 0 throttle.
3. A visible **EMERGENCY STOP** is always on screen and on a keyboard shortcut. It sends throttle 0 immediately, bypassing any queue or ramp.
4. **Heartbeat/watchdog:** if telemetry or the UI link stops for more than ~500 ms (TODO tune), command throttle 0. The firmware should do this on its own too, not rely only on the app.
5. **Configurable hard limits**, checked on every sample, trip -> stop + latch fault until the user clears it:
   max current, max voltage and min voltage (battery cutoff), max temperature, max RPM, max thrust (load-cell rating).
   Limits must be set before the first run; no silent defaults of "unlimited".
6. Ramp throttle with a bounded slew rate; reject step jumps above a configured size unless the test type is explicitly a step test.
7. Sensor sanity: flag stale, NaN, out-of-range or stuck readings and treat them as a fault, never as zero.
8. Never auto-run a test on page load, route change, or hot reload.

## Test types
- **Step**: hold throttle levels for fixed dwell time, record steady-state (discard the settling window).
- **Ramp**: slow linear throttle sweep, good for a thrust/efficiency curve.
- **Sweep / matrix**: motor x prop x voltage combinations, each as its own run.
- **Endurance**: long hold; log temperature drift, stop on thermal limit.
- **Dynamic response**: fast steps to measure spin-up/spin-down time.

Each run stores: motor, prop, ESC, battery, ambient temp/pressure, calibration used, firmware/app version, start/end time, abort reason.

## Data handling
- Sample at a fixed rate with a monotonic timestamp (ms since run start) from the device, not the browser clock.
- Raw samples are append-only; filtering/averaging is a view, not a mutation. Keep raw and filtered separate.
- Calibration (load-cell scale and tare, current offset, pole count) is versioned and attached to every run. Tare before each run.
- Export CSV with a header row, SI units in column names (`thrust_N`, `current_A`), one row per sample, plus a metadata block or sidecar JSON.
- Decimate for charts (e.g. LTTB), never for stored/exported data.

## UI guidelines
- Dashboard shows live thrust, RPM, current, voltage, power, temperature, and arm/fault state at a glance; fault state is unmistakable (color + text + icon).
- Primary charts: thrust vs throttle, thrust vs RPM, efficiency (g/W) vs thrust, current vs thrust. Label axes with units.
- Disable motor-spinning controls unless connected + armed + no fault.
- Use the `frontend-design` and `dataviz` skills for look and charts; keep the live view readable at a distance on a bench monitor.

## Testing
- Build a **simulated device** (fake serial/WebSocket source) so the UI and logic are testable without hardware or spinning motors; use it with `webapp-testing`.
- Unit-test: unit conversions, efficiency math and zero-division guards, limit tripping, watchdog timeout, ramp slew limiting, CSV export.
- Any change touching throttle output or limits needs a test proving the safe state (throttle 0) on fault, disconnect, and startup.
- Use `test-driven-development` and `verification-before-completion`; do not claim safety behavior works without running the tests.
