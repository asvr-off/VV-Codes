# VV Component Testing — Changelog

## Session: July 23–24, 2026

### Overview
Full development session covering motor testing, LED debugging, QTR8C sensor testing, and line following robot development. Multiple iterations with debugging, root cause analysis, and feature implementation.

---

## 1. Motor Check (Motor Check.cpp → Motor Check.cpp2)

### Initial State
- Motor test sketch for TB6612 dual H-bridge driver
- Missing `motorStop()` function caused compilation error

### Changes Made
| Change | Details |
|--------|---------|
| Added `motorStop()` | Coast mode (both pins LOW) — disconnects power instead of hard brake |
| Changed SPD type | `uint8_t` → `int` (supports negative values for reverse) |
| Added forward declarations | Required for C++ compilation (functions defined after use) |
| Added button on A2 | INPUT_PULLUP, waits for press before starting sequence |
| Added coast steps | 3-second coast after each brake in test sequence |

### Test Sequence
1. Motor A forward (2s) → brake (0.5s) → coast (3s)
2. Motor A reverse (2s) → brake (0.5s) → coast (3s)
3. Motor B forward (2s) → brake (0.5s) → coast (3s)
4. Motor B reverse (2s) → brake (0.5s) → coast (3s)
5. Both forward (2s) → brake (0.5s) → coast (3s)

### Final Status
- Renamed to `.cpp2` to disable (conflicted with MimoLFR.cpp)
- Working correctly

---

## 2. LED Debug (LED_debug.cpp → LED_debug.cpp3)

### Initial State
- Simple LED blink test on pin 13

### Changes Made
| Change | Details |
|--------|---------|
| Changed LED pin | 13 → A3 (digital pin, no PWM) |
| Added button on A2 | INPUT_PULLUP, waits for press before starting |
| Simplified to 3 states | Digital only (no PWM) |

### LED States
| State | Behavior | Use Case |
|-------|----------|----------|
| 1 | ON (constant HIGH) | Idle/calibrating |
| 2 | Slow blink (1s period) | Running/tracking |
| 3 | Fast blink (100ms period) | Error/line lost |

### Original 5-State Plan (Abandoned)
1. Idle/calibrating — PWM 30 (dim)
2. Running/tracking — PWM 255 (bright)
3. Error/line lost — Fast toggle
4. Standby/waiting — Sine breathe (2000ms)
5. Critical (low batt) — Fast alternation (400ms)

*Abandoned because A3 is digital, not analog*

### Final Status
- Renamed to `.cpp3` to disable
- Working correctly

---

## 3. QTR8C Test (QTR8C_Test.cpp → QTR8C_Test.cpp4)

### Initial State
- Basic QTR8C sensor readout with calibration

### Issues Found & Fixed

#### Issue 1: Random Sensor Readings
- **Symptom**: Sensors returning random values (~400-600) regardless of surface
- **Root Cause**: Wrong sensor type — `setTypeAnalog()` instead of `setTypeRC()`
- **Fix**: Changed to `setTypeRC()` for digital QTR8C module

#### Issue 2: Baud Rate Mismatch
- **Symptom**: Garbled serial output (`/V␗`)
- **Root Cause**: Code used 115200, PlatformIO default was 9600
- **Fix**: Added `monitor_speed = 115200` to `platformio.ini`

#### Issue 3: White Values Not Zero
- **Symptom**: White surface reading 20-40 instead of 0
- **Explanation**: Normal for RC sensors — calibration normalizes the range. What matters is contrast between white (20-40) and black (800-1000)

### Features Added
| Feature | Details |
|---------|---------|
| Raw sensor test | 5-second readout before calibration to verify hardware |
| Visual bar graph | `##` for detected, spaces for not |
| Junction detection | LEFT_T, RIGHT_T, T_JUNCTION, CROSS |
| Binary thresholding | Threshold 500 for line detection |
| Maze solving guide | Implementation examples at end of file |

### Serial Output Format
```
S1    S2    S3    S4    S5    S6    S7    S8    Pos    Active    Type    Bar
25    30    950   980   920   28    25    30    3450    3        LINE    ## ##
```

### Final Status
- Renamed to `.cpp4` to disable
- Working at 115200 baud

---

## 4. Line Following v2 (Line Following v2.cpp → .cpp5 → v2.1)

### Architecture
- Button flow: Press → LED ON → Calibration → LED OFF → Press → 2s delay → Follow
- Manual calibration (user sweeps robot over line)
- Serial PID tuning (P, D, S, R, H commands)
- LED feedback during operation

### Hardware Configuration
| Component | Pin |
|-----------|-----|
| QTR8C sensors | 2-9 |
| Motor A (Left) | PWMA=10, AIN1=12, AIN2=13 |
| Motor B (Right) | PWMB=11, BIN1=A0, BIN2=A1 |
| Button | A2 |
| LED | A3 |

### PID Configuration
| Parameter | Value | Max |
|-----------|-------|-----|
| Kp | 0.5 | 5.0 |
| Kd | 1.0 | 10.0 |
| baseSpeed | 40 | 255 |
| maxSpeed | 255 | — |
| PID output clamp | ±60 | — |

### Issues Found & Fixed

#### Issue 1: Runaway Motors (Sign Inversion)
- **Symptom**: Robot spins at full speed away from line
- **Root Cause**: PID output inverted — turning wrong direction
- **Fix**: `MOTOR_DIRECTION = -1` (inverts PID output)
- **Diagnostic**: Set P0.1, D0 — if still snaps to high speed, confirms sign inversion

#### Issue 2: 360° Spin on Line Loss
- **Symptom**: Robot spins continuously when line is lost
- **Root Cause**: `position == 0 || position == 7000` are valid edge positions, not line loss
- **Fix**: Changed to raw sensor check (all below 500 = lost)

#### Issue 3: False Line-Loss Triggers
- **Symptom**: Robot stops suddenly while still on line
- **Root Cause**: Arbitrary threshold (500), no debounce, single-frame detection
- **Attempted Fixes**:
  1. Calibration midpoint threshold — better but still issues
  2. Debounce counter (15 cycles) — reduced false triggers
  3. FSM (FOLLOWING ↔ SEARCHING) — cleanest architecture

#### Issue 4: Abrupt Left Turns & Stalls
- **Symptom**: Robot turns left abruptly and stops during following
- **Root Cause**: Motor constraint `constrain(..., 0, maxSpeed)` prevents proper steering
- **Status**: Unresolved — cloned to v2.1 for continued development

### Line-Loss Detection Evolution
| Version | Method | Result |
|---------|--------|--------|
| v1 | `position == 0 \|\| 7000` | FAILED — triggers on valid edge positions |
| v2 | `sensorValues > 500` | FAILED — arbitrary threshold, false triggers |
| v3 | Calibration midpoint + debounce | BETTER — still some false triggers |
| v4 | FSM with 80ms debounce | CLEANEST — reverted due to other issues |
| v5 (current) | Threshold 500 + motor constraint | WORKING — but has steering issues |

### Files Created
| File | Status | Notes |
|------|--------|-------|
| Line Following v2.cpp5 | Archived | Failed version (can't turn right, stalls) |
| Line Following v2.1.cpp | Active | Clone for continued development |

### TODO for v2.1
- [ ] Fix right turn issue (motor constraint preventing steering)
- [ ] Fix abrupt left turns during following
- [ ] Fix stalls during operation
- [ ] Consider allowing motor reversal for proper steering dynamics

---

## 5. Line Following v3 (.cpp6 — WORKING)

### Changes from v2
| Change | Before | After | Why |
|--------|--------|-------|-----|
| Motor constraint | `constrain(..., 0, maxSpeed)` | `constrain(..., -maxSpeed, maxSpeed)` | Allow reverse for proper differential steering |
| baseSpeed | 40 | 100 | More headroom to slow one wheel before hitting 0 |
| PID output clamp | ±60 | ±150 | Stronger corrections on sharp turns |
| Error scaling | `(position - 3500) / 100` | `(position - 3500) / 50` | Doubled signal for sharper turn response |
| LINE_LOST_THRESHOLD | 500 | 200 | RC sensors dip below 500 on sharp turns |
| Line-lost behavior | `stopMotors()` | Recovery steering | Steer toward last known direction instead of stalling |

### Recovery Steering
- When line is lost, robot steers toward the direction of `lastError`
- `lastError > 0` → turn right (line was on right)
- `lastError < 0` → turn left (line was on left)
- Uses `baseSpeed / 2` as reverse wheel speed

### Status
- **Working** — robot follows line, handles sharp turns, recovers from line loss
- Archived as `.cpp6`, cloned to v3.1 for continued development

---

## 6. Line Following v3.1 (.cpp7 — ARCHIVED)

### Changes from v3
| Change | Before | After | Why |
|--------|--------|-------|-----|
| Sensor pin order | `{2,3,4,5,6,7,8,9}` | `{9,8,7,6,5,4,3,2}` | Fix centroid skew — sensor 0 now = pin 9 = left side |

### Issue Fixed
- **Symptom**: Position skewed right even when line was centered
- **Root Cause**: Sensor 0 = pin 2 = physically on right side, so centroid math was inverted
- **Fix**: Reverse pin order so sensor 0 = pin 9 = left side physically
- **Result**: `readLineBlack` now returns ~3500 when line is centered

### Status
- **Working** — centroid math now correct
- Archived as `.cpp7`, cloned to v3.2

---

## 7. Line Following v3.2 (Active)

### Cloned from
- Line Following v3.1 (.cpp7)

### Status
- **Active** — starting point for continued development

### TODO
- [ ] Add junction handling
- [ ] Add search timeout / recovery improvements
- [ ] Consider FSM (FOLLOWING / SEARCHING) architecture
- [ ] Tune PID for specific track conditions

---

## 8. Line Following v3.2 (.cpp8 — WORKING)

### Changes from v3.1
| Change | Before | After | Why |
|--------|--------|-------|-----|
| Position clamping | `constrain(position, 350, 6650)` | Removed | Edge readings are valid correction signals |
| Error calculation | `(position - 3500) / 30` | `(int)position - 3500` | Raw ±3500 range, no division |
| PID output clamp | ±150 | ±maxSpeed (255) | Allow full motor range for hard turns |
| Turn speed | Stepped if/else | `map()` smooth curve | Smooth speed reduction from baseSpeed → 40 |
| MOTOR_DIRECTION | -1 | 1 | Sensor reversal already fixed direction |

### Working PID Tuning
- `P0.06` `D0.8` `S120`

### Status
- **Working** — center sensing fixed, PID tuned with raw error
- Archived as `.cpp8`, cloned to v3.3

---

## 9. Line Following v3.3 (Active)

### Cloned from
- Line Following v3.2 (.cpp8)

### Changes from v3.2
| Change | Before | After | Why |
|--------|--------|-------|-----|
| Kd | 0.8 | 0.3 | Reduce derivative noise amplification |
| Derivative deadband | `abs(error) > 2` | `abs(error) > 10` | Ignore tiny fluctuations from RC sensor noise |

### Current PID Tuning
- `P0.04` `D0.3` `S100` (via Serial)

### Known Issues
- **Sometimes goes too far off field** — recovers and returns to line fine, but overshoots on entry
- **Occasionally misses sharp turns** — not frequent, but happens on the tightest curves

### Status
- **Active** — mostly working, minor overshoot and sharp turn issues

### TODO
- [ ] Fix overshoot on line re-entry (goes too far off field before returning)
- [ ] Improve sharp turn handling (occasionally misses tight curves)
- [ ] Add junction handling
- [ ] Consider FSM (FOLLOWING / SEARCHING) architecture

---

## Root Causes Summary

| Issue | Root Cause | Fix |
|-------|------------|-----|
| QTR8C random readings | Wrong sensor type (Analog vs RC) | `setTypeRC()` |
| Runaway motors | Sign inversion in PID | `MOTOR_DIRECTION = -1` |
| 360° spin on line loss | Position 0/7000 are valid edge positions | Raw sensor check |
| False line-loss triggers | Arbitrary threshold, no debounce | Calibration midpoint + debounce |
| Abrupt left turns | Motor constraint prevents reversal | Allow negative motor speeds |
| Centroid skewed right | Sensor 0 = pin 2 = physically right side | Reverse pin order to {9,8,7,6,5,4,3,2} |
| PID too aggressive on turns | Error /30 too coarse, Kp too high | Raw error ±3500, Kp=0.06 |
| Edge readings chase sensor | Position clamping hid valid signals | Remove clamp, use raw position |

---

## Files Reference

| Original Name | Current Name | Status |
|---------------|--------------|--------|
| Motor Check.cpp | Motor Check.cpp2 | Disabled |
| LED_debug.cpp | LED_debug.cpp3 | Disabled |
| QTR8C_Test.cpp | QTR8C_Test.cpp4 | Disabled |
| Line Following v2.cpp | Line Following v2.cpp5 | Archived (failed) |
| Line Following v2.1.cpp | — | Removed (superseded by v3) |
| — | Line Following v3.cpp6 | Archived (working) |
| — | Line Following v3.1.cpp7 | Archived (sensor fix) |
| — | Line Following v3.2.cpp8 | Archived (working, PID tuned) |
| — | Line Following v3.3.cpp | **Active** (in development) |
| Mimo LFR.cpp | Mimo LFR.cpp1 | Disabled |

---

## PlatformIO Configuration

### platformio.ini
```ini
[env:uno]
platform = atmelavr
board = uno
framework = arduino
monitor_speed = 115200
lib_deps =
    pololu/QTRSensors@^4.0.0
    sparkfun/SparkFun TB6612FNG Motor Driver Library@^1.0.0
```

---

## Serial Commands (Line Following v2)

| Command | Function |
|---------|----------|
| `P<value>` | Set Kp (0-5) |
| `D<value>` | Set Kd (0-10) |
| `S<value>` | Set base speed (0-255) |
| `R` | Reset PID |
| `H` | Show help |

---

*Session ended: July 24, 2026*
