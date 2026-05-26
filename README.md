# SlimeVR-Tracker-nRF

[![CI](https://github.com/SlimeVR/SlimeVR-Tracker-nRF/actions/workflows/workflow.yml/badge.svg)](https://github.com/SlimeVR/SlimeVR-Tracker-nRF/actions/workflows/workflow.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE-MIT)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE-APACHE)

> Wireless VR motion tracker firmware for the SlimeVR ecosystem, targeting Nordic Semiconductor **nRF52** and **nRF54L** series SoCs.

## Overview

**SlimeVR-Tracker-nRF** (also known as **SlimeNRF**) is an open-source firmware for low-latency, wireless IMU trackers used in VR full-body tracking. It runs on Zephyr RTOS and leverages Nordic's **Enhanced Shock Burst (ESB)** protocol for robust 2.4 GHz communication.

- **Repository**: [SlimeVR/SlimeVR-Tracker-nRF](https://github.com/SlimeVR/SlimeVR-Tracker-nRF)
- **Documentation**: [https://docs.slimevr.dev/smol-slimes](https://docs.slimevr.dev/smol-slimes)
- **Hardware PCB**: [SlimeVR-Tracker-nRF-PCB](https://github.com/SlimeVR/SlimeVR-Tracker-nRF-PCB)
- **License**: Dual-licensed under MIT and Apache-2.0

## Supported Hardware

### Development Kits
| Board | SoC | Format |
|-------|-----|--------|
| `nrf52840dk/nrf52840` | nRF52840 | `.hex` |
| `nrf52dk/nrf52832` | nRF52832 | `.hex` |
| `nrf5340dk/nrf5340/cpuapp` | nRF5340 | `.hex` |
| `nrf54l15dk/nrf54l15/cpuapp` | nRF54L15 | `.hex` |
| `nrf21540dk/nrf52840` | nRF52840 + FEM | `.hex` |

### SlimeVR Official Boards
| Board | SoC | Format |
|-------|-----|--------|
| `slimenrf_r1/nrf52840` | nRF52840 | `.hex` |
| `slimenrf_r2/nrf52832` | nRF52832 | `.hex` |
| `slimenrf_r3/nrf52840` | nRF52840 | `.uf2` |
| `slimevrmini_p1_uf2/nrf52833` | nRF52833 | `.uf2` |
| `slimevrmini_p2_uf2/nrf52833` | nRF52833 | `.uf2` |
| `slimevrmini_p3r6_uf2/nrf52833` | nRF52833 | `.uf2` |
| `slimevrmini_p3r7_uf2/nrf52833` | nRF52833 | `.uf2` |
| `slimevrmini_p4_uf2/nrf52833` | nRF52833 | `.uf2` |
| `slimevrmini_p4r9_uf2/nrf52833` | nRF52833 | `.uf2` |
| `slimevrmini_p4r11_uf2/nrf52833` | nRF52833 | `.uf2` |

### Community Boards
| Board | SoC | Bus | Format |
|-------|-----|-----|--------|
| `promicro_uf2/nrf52840/i2c` | nRF52840 | I2C | `.uf2` |
| `promicro_uf2/nrf52840/spi` | nRF52840 | SPI | `.uf2` |
| `promicro_uf2/nrf52840/smspi` | nRF52840 | smSPI | `.uf2` |
| `promicro_uf2/nrf52840/chrysalis` | nRF52840 | I2C | `.uf2` |
| `xiao_ble/nrf52840` | nRF52840 | I2C | `.uf2` |
| `xiao_ble/nrf52840/sense` | nRF52840 | I2C | `.uf2` |
| `mochi` (Mocha-Elec) | nRF52840 | — | `.uf2` |

## Supported Sensors

### IMUs
| Sensor | Driver | Notes |
|--------|--------|-------|
| **BMI270** | `BMI270.c` | Bosch; requires firmware blob |
| **ICM-42688-P / ICM-42688-V** | `ICM42688.c` | TDK InvenSense |
| **ICM-45686 / ICM-45688-P** | `ICM45686.c` | TDK InvenSense |
| **ISM330BX** | `ISM330BX.c` | ST (treated as LSM6DSV variant on server) |
| **LSM6DSM** | `LSM6DSM.c` | ST (LSM6DS3TR-C family compatible) |
| **LSM6DSO** | `LSM6DSO.c` | ST |
| **LSM6DSV** | `LSM6DSV.c` | ST |

### Magnetometers
AK09940, BMM150, BMM350, IST8306, IST8308, LIS2MDL, LIS3MDL, MMC5983MA, QMC6309

## Architecture Highlights

- **RTOS**: Zephyr (via nRF Connect SDK v3.1-branch)
- **Wireless**: Nordic ESB (2.4 GHz proprietary) with automatic pairing
- **Sensor Fusion**: VQF (Kalman-based, default) or x-io Fusion (Madgwick-based, optional)
- **Power Management**: Active → Low Power → IMU Wake-Up → User Shutdown
- **USB**: HID data output + CDC-ACM console
- **Calibration**: 6-side accelerometer calibration, gyro bias, magnetometer hard/soft iron

## Building from Source

### Prerequisites
- [Zephyr SDK](https://github.com/zephyrproject-rtos/sdk-ng/releases) (CI uses v0.16.9)
- Python 3 with `west` and `ninja`
- nRF Connect SDK (managed via `west.yml`, currently `v3.1-branch`)

### Workspace Setup
```bash
west init -l SlimeVR-Tracker-nRF
west update --narrow -o=--depth=1
west zephyr-export
pip install -r zephyr/scripts/requirements.txt
```

### Build
```bash
# Example: ProMicro I2C
west build \
  --board promicro_uf2/nrf52840/i2c \
  --pristine=always SlimeVR-Tracker-nRF \
  -- -DBOARD_ROOT=./SlimeVR-Tracker-nRF

# Example: XIAO BLE Sense
west build \
  --board xiao_ble/nrf52840/sense \
  --pristine=always SlimeVR-Tracker-nRF \
  -- -DBOARD_ROOT=./SlimeVR-Tracker-nRF
```

Output: `build/SlimeVR-Tracker-nRF/zephyr/zephyr.uf2` or `zephyr.hex`

## Project Structure

```
SlimeVR-Tracker-nRF/
├── src/
│   ├── main.c                 # Boot logic, reset handling, button/DFU
│   ├── console.c/h            # Shell console, settings, calibration UI
│   ├── config.c/h             # NVS persistent settings (4 config groups)
│   ├── globals.h              # Shared macros: sensor alignment, quaternion correction
│   ├── hid.c/h                # USB HID data output
│   ├── connection/            # Wireless communication
│   │   ├── esb.c/h            # Enhanced Shock Burst (ESB) radio
│   │   ├── timer.c/h          # Radio timing utilities
│   │   └── connection.c/h     # Connection state management
│   ├── sensor/                # IMU / magnetometer / sensor fusion
│   │   ├── sensor.c/h         # Main sensor thread, data pipeline
│   │   ├── interface.c/h      # Generic sensor interface abstraction
│   │   ├── calibration.c/h    # 6-side accel calibration, gyro bias
│   │   ├── scan.c/h           # I2C device auto-detection
│   │   ├── scan_spi.c/h       # SPI device auto-detection
│   │   ├── imu/               # IMU drivers
│   │   ├── mag/               # Magnetometer drivers
│   │   └── fusion/            # Sensor fusion algorithms (VQF, etc.)
│   └── system/                # Board support / power / status
│       ├── system.c/h         # System init, retained memory, die temp
│       ├── power.c/h          # Power management, DCDC/LDO, shutdown
│       ├── battery.c/h        # Battery voltage monitoring, SOC estimation
│       ├── led.c/h            # Status LED patterns / priorities
│       └── status.c/h         # System status flags
├── boards/                    # Custom board definitions (DTS, defconfig, board.yml)
│   ├── nordic/                # promicro_uf2, test54l, test805
│   ├── slimevr/               # slimennrf_r1/r2/r3, slimevrmini_p*
│   ├── mocha-elec/            # mochi
│   └── sctanf/
├── socs/                      # SoC-level DeviceTree overlays
├── pm_static/                 # Partition Manager static configs
├── vqf-c/                     # VQF sensor fusion library (external)
├── west.yml                   # West manifest (nRF Connect SDK v3.1-branch)
├── prj.conf                   # Default Kconfig project settings
├── Kconfig                    # Application-level Kconfig symbols
└── .github/workflows/         # CI builds for all supported boards
```

## Configuration

Key Kconfig options (see `Kconfig` and `prj.conf`):

| Option | Default | Description |
|--------|---------|-------------|
| `SENSOR_USE_VQF` | `y` | Sensor fusion algorithm (VQF) |
| `SENSOR_USE_MAG` | `y` | Enable magnetometer if present |
| `USE_IMU_WAKE_UP` | `y` | IMU wake-up motion detection |
| `USER_SHUTDOWN` | `y` | Allow user shutdown via button/reset |
| `SENSOR_ACCEL_ODR` | `100` | Accelerometer ODR (Hz) |
| `SENSOR_GYRO_ODR` | `200` | Gyroscope ODR (Hz) |
| `RADIO_TX_POWER` | `8` | Radio TX power (dBm) |
| `CONNECTION_OVER_HID` | `n` | Use USB HID as primary data output |

## Flashing

### UF2 Bootloader (drag & drop)
1. Double-tap the reset button to enter bootloader mode.
2. A USB mass storage device appears.
3. Copy `zephyr.uf2` to the drive.

### nRF Connect Programmer / `nrfjprog`
For `.hex` files on DKs or custom boards without a UF2 bootloader:
```bash
nrfjprog --program zephyr.hex --sectorerase --reset
```

## Contributing

1. Fork the repository and create a feature branch.
2. Follow the existing coding style (C, Zephyr APIs, `LOG_MODULE_REGISTER` for logging).
3. Ensure all SlimeVR-original files include the MIT license header.
4. Add your board to the CI matrix in `.github/workflows/workflow.yml` if applicable.
5. Submit a Pull Request.

## Acknowledgments

- [SlimeVR](https://slimevr.dev/) community and contributors
- [Nordic Semiconductor](https://www.nordicsemi.com/) for nRF Connect SDK and Zephyr integration
- [Zephyr Project](https://zephyrproject.org/) for the RTOS foundation
- VQF sensor fusion library authors

---

*For detailed hardware guides and tracker assembly instructions, visit the [SlimeVR Docs](https://docs.slimevr.dev/smol-slimes).*
