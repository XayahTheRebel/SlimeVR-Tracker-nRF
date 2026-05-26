# SlimeVR-Tracker-nRF 开发指南

> **重要规则：每次修改文件后都必须执行 `git add` 和 `git commit` 提交更改。**
> 
> 不要累积多次修改后再一次性提交。每完成一个逻辑单元的修改（例如：修复一个 bug、添加一个功能、更新一份文档），立即提交。这有助于保持清晰的修改历史，方便回滚和代码审查。
>
> **执行任务前，先查看当前可用的 Skills。** 在 `~/.agents/skills/`（User 作用域）和项目内置 Skills 中，可能有与任务相关的专用能力（如 `diagnose`、`tdd`、`to-issues` 等）。阅读对应 `SKILL.md` 可以获取最佳实践、专用工具链和决策模板，避免重复造轮子。

## 项目概述

本项目是 **SlimeNRF** 追踪器的固件，用于 SlimeVR 生态系统的无线 VR 动作追踪器。目标平台为 Nordic Semiconductor **nRF52** 和 **nRF54L** 系列 SoC，基于 Zephyr RTOS 和 nRF Connect SDK (v3.1-branch) 构建。

- **仓库**: SlimeVR/SlimeVR-Tracker-nRF
- **许可证**: 双许可证 MIT / Apache-2.0
- **文档**: https://docs.slimevr.dev/smol-slimes
- **硬件 PCB**: https://github.com/SlimeVR/SlimeVR-Tracker-nRF-PCB

## 构建系统

### 前置依赖
- Zephyr SDK (CI 中使用 v0.16.9)
- Python 3 + west + ninja
- nRF Connect SDK (通过 `west.yml` 管理，当前为 `v3.1-branch`)

### 工作区初始化
```bash
west init -l SlimeVR-Tracker-nRF
west update --narrow -o=--depth=1
west zephyr-export
pip install -r zephyr/scripts/requirements.txt
```

### 编译
```bash
# 示例：为 ProMicro I2C 板子编译
west build --board promicro_uf2/nrf52840/i2c --pristine=always SlimeVR-Tracker-nRF -- -DBOARD_ROOT=./SlimeVR-Tracker-nRF

# 其他常用板子：
# - promicro_uf2/nrf52840/spi
# - xiao_ble/nrf52840
# - xiao_ble/nrf52840/sense
# - slimennrf_r1/nrf52840
# - slimennrf_r2/nrf52832
# - nrf52840dk/nrf52840
# - nrf52dk/nrf52832
```

编译输出：`build/SlimeVR-Tracker-nRF/zephyr/zephyr.uf2` 或 `.hex`。

## 代码架构

### 目录结构
```
src/
  main.c              # 入口点、启动逻辑、复位处理、按键/DFU
  console.c           # Shell 控制台、设置命令、校准界面
  config.c/h          # NVS 持久化设置（4 个配置组）
  globals.h           # 共享宏：传感器轴对齐、四元数校正
  hid.c/h             # USB HID 数据输出
  parse_args.c/h      # 参数解析辅助函数
  retained.c/h        # 跨复位保留的内存
  build_defines.h     # 版本宏、服务器常量（IMU/MCU/板子 ID）

  connection/         # 无线通信
    esb.c/h           # Enhanced Shock Burst（Nordic 专有 2.4GHz 射频）
    timer.c/h         # 射频定时工具
    connection.c/h    # 连接状态管理

  sensor/             # IMU / 磁力计 / 传感器融合
    sensor.c/h        # 主传感器线程、数据流水线
    interface.c/h     # 通用传感器接口抽象
    calibration.c/h   # 6 面加速度计校准、陀螺仪零偏
    sensors_enum.h    # IMU 和磁力计设备枚举 + ID 查找表
    scan.c/h          # I2C 设备探测
    scan_spi.c/h      # SPI 设备探测
    scan_ext.c/h      # 扩展/辅助总线扫描
    sensor_none.c/h   # 空传感器回退

    imu/              # IMU 驱动
      BMI270.c/h, ICM42688.c/h, ICM45686.c/h, ISM330BX.c/h,
      LSM6DSM.c/h, LSM6DSO.c/h, LSM6DSV.c/h

    mag/              # 磁力计驱动
      AK09940.c/h, BMM150.c/h, BMM350.c/h, IST8306.c/h,
      IST8308.c/h, LIS2MDL.c/h, LIS3MDL.c/h, MMC5983MA.c/h, QMC6309.c/h

    fusion/           # 传感器融合算法
      fusion_none.c/h
      vqf/vqf.c/h     # VQF（基于卡尔曼滤波）— 默认
      # x-io fusion 可通过 Kconfig 选择，但不在本仓库中

    magneto/          # Magneto 校准算法（1.4）
      magneto1_4.c/h
      mymathlib_matrix.c/h

  system/             # 板级支持 / 电源 / 状态
    system.c/h        # 系统初始化、保留内存、芯片温度、感应引脚
    power.c/h         # 电源管理、DCDC/LDO、关机
    battery.c/h       # 电池电压监测、电量估算
    battery_tracker.c/h
    led.c/h           # 状态 LED 模式/优先级
    status.c/h        # 系统状态标志
    rtt_console.h

boards/               # 自定义板子定义（DTS、defconfig、board.yml）
  nordic/             # promicro_uf2、test54l
  slimevr/            # slimennrf_r1/r2/r3、slimevrmini_p*
  mocha-elec/         # mochi
  sctanf/

socs/                 # SoC 级设备树 overlay
pm_static/            # 分区管理器静态配置
```

### 核心概念

**传感器对齐约定** (`globals.h`)：
- 平放 / 面朝上时：左侧（设备视角）为 `+X`，正面（朝上）为 `+Z`
- 穿戴在身体上 / 竖直时：设备顶部为 `+Y`，正面（朝外）为 `+Z`
- `SENSOR_QUATERNION_CORRECTION` 按板子校正传感器到安装方向。

**配置系统** (`config.c/h`)：
- 4 组配置存储在 NVS 中：
  - `config_0`：设备设置（位标志）— 例如用户关机、IMU 唤醒
  - `config_1`：传感器设置（位标志）— 例如低功耗、磁力计启用
  - `config_2`：int16 设置数组 — LED 颜色、ODR、量程、融合模式、发射功率
  - `config_3`：int32 设置数组 — 超时时间、阈值
- 通过 `CONFIG_0_SETTINGS_READ(id)`、`CONFIG_2_SETTINGS_READ(id)` 等访问。

**电源状态**：
- Active（活跃）：正常追踪
- Low Power（低功耗，`SENSOR_USE_LOW_POWER_2`）：无运动时进入高延迟传感器模式
- IMU Wake Up（IMU 唤醒，`USE_IMU_WAKE_UP`）：最低功耗，运动中断唤醒设备
- User Shutdown（用户关机）：完全断电，按键或底座唤醒

**连接方式**：
- 主通道：ESB（Enhanced Shock Burst）— Nordic 专有 2.4GHz
- 备用/辅助：USB HID（`CONFIG_0_CONNECTION_OVER_HID`）

**传感器融合**：
- 默认：VQF（基于卡尔曼滤波），通过 `vqf-c/` 外部库实现
- 备选：x-io Fusion（基于 Madgwick 滤波），可通过 Kconfig 选择

## 编码规范

- **语言**：C（嵌入式），使用 Zephyr API
- **头文件**：使用 `#ifndef SLIMENRF_xxx` 包含保护
- **许可证头**：所有 SlimeVR 原创文件必须带 MIT 许可证声明
- **日志**：使用 Zephyr `LOG_MODULE_REGISTER(name, level)` + `LOG_INF/WRN/ERR`
- **线程**：Zephyr k_threads；主传感器逻辑在独立线程中运行
- **延时**：使用 `k_msleep()` / `k_uptime_get()` — 禁止忙等待
- **设置项**：通过 Kconfig 暴露可调参数，禁止硬编码魔法数字
- **设备树**：硬件配置主要通过设备树（`*.dts`、`*.overlay`）完成

## 硬件支持

### 支持的 IMU
| IMU | 驱动 | 说明 |
|-----|------|------|
| BMI270 | `BMI270.c` | Bosch，需要固件 blob（`BMI270_firmware.h`） |
| ICM-42688-P/V | `ICM42688.c` | TDK InvenSense |
| ICM-45686/688 | `ICM45686.c` | TDK InvenSense |
| ISM330BX | `ISM330BX.c` | ST（在服务器中被当作 LSM6DSV 变体） |
| LSM6DSM | `LSM6DSM.c` | ST（LSM6DS3TR-C 家族） |
| LSM6DSO | `LSM6DSO.c` | ST |
| LSM6DSV | `LSM6DSV.c` | ST |

### 支持的磁力计
AK09940、BMM150、BMM350、IST8306、IST8308、LIS2MDL、LIS3MDL、MMC5983MA、QMC6309

## 添加新板子

1. 在 `boards/<厂商>/<板子>/` 下创建：
   - `board.yml` — 板子元数据
   - `<板子>.dts` — 设备树（引脚、总线、传感器、LED）
   - `<板子>_defconfig` — 默认 Kconfig 值
   - `Kconfig.<板子>` — 板子专属 Kconfig 符号
2. 如需特殊安装方向，在 `globals.h` 中添加 `SENSOR_QUATERNION_CORRECTION` 和 `SENSOR_MAGNETOMETER_AXES_ALIGNMENT`。
3. 在 `.github/workflows/workflow.yml` 的 CI 矩阵中添加新板子。
4. 如果使用 UF2 bootloader，在 `pm_static/` 中添加分区配置。

## 添加新 IMU

1. 创建 `src/sensor/imu/<芯片>.c` 和 `.h`，实现 `src/sensor/interface.h` 中定义的接口。
2. 在 `sensors_enum.h` 中添加芯片 ID（`enum dev_imu`）。
3. 在 `build_defines.h` 中添加服务器常量映射（`get_server_constant_imu_id`）。
4. 在 `scan.c`（I2C）和/或 `scan_spi.c`（SPI）中添加探测逻辑。
5. 如需，为驱动添加 Kconfig 符号。

## 常见陷阱

- **时钟配置**：`CONFIG_CLOCK_CONTROL_NRF_K32SRC_RC=y` 可加快启动，但可能降低时钟精度。
- **栈大小**：`CONFIG_MAIN_STACK_SIZE=512`；在 nRF54L 上若崩溃可能需要调整。
- **I2C 超时**：`CONFIG_I2C_NRFX_TRANSFER_TIMEOUT=25`（ms）— 足够传输约 1000 字节。
- **UF2 bootloader**：双击复位行为在 `main.c` 中通过保留内存（`DFU_DBL_RESET_MEM`）管理。
- **传感器融合**：VQF 需要 `CONFIG_FPU=y`。
- **复位处理**：重启计数器（100–200）用于区分复位原因，触发配对或关机。
