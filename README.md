# VV Component Testing

Line following robot development using Arduino Uno, QTR-8C sensor array, and TB6612FNG motor driver.

## Hardware

| Component | Details |
|-----------|---------|
| Microcontroller | Arduino Uno |
| Sensors | QTR-8C Reflectance Sensor Array (8 sensors) |
| Motor Driver | TB6612FNG Dual H-Bridge |
| Motors | 2x DC gear motors |

### Pin Connections

| Pin | Function |
|-----|----------|
| 2-9 | QTR-8C sensors (reversed order for correct centroid math) |
| 10 | PWMA (Left motor speed) |
| 11 | PWMB (Right motor speed) |
| 12, 13 | AIN1, AIN2 (Left motor direction) |
| A0, A1 | BIN1, BIN2 (Right motor direction) |
| A2 | Start button (INPUT_PULLUP) |
| A3 | Status LED |

## Features

- Manual sensor calibration (4-second sweep)
- PD line following with raw position error
- Recovery steering on line loss (steers toward last known direction)
- Serial PID tuning (P, D, S, R, H commands)
- LED status feedback (OFF = centered, Blink = correcting, Solid = line lost)
- Adaptive speed reduction on hard turns

## Files

| File | Status | Description |
|------|--------|-------------|
| `Line Following v3.3.cpp` | Active | Current working version |
| `Line Following v3.2.cpp8` | Archived | PID tuned with raw error |
| `Line Following v3.1.cpp7` | Archived | Sensor pin reversal fix |
| `Line Following v3.cpp6` | Archived | Motor constraint & recovery steering |
| `Line Following v2.cpp5` | Archived | Failed version (steering issues) |
| `QTR8C_Test.cpp4` | Test | Sensor readout & junction detection |
| `Motor Check.cpp2` | Test | Motor driver test sequence |
| `LED_debug.cpp3` | Test | LED state test |

## Serial Commands

| Command | Function |
|---------|----------|
| `P<value>` | Set Kp (0-5) |
| `D<value>` | Set Kd (0-10) |
| `S<value>` | Set base speed (0-255) |
| `R` | Reset PID |
| `H` | Show help |

## Build

PlatformIO with Arduino framework:

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

## License

MIT
