# N16R8 硬件手册

ESP32-S3 N16R8（GOOUUU 板）是 ESP-Cam 家族里唯一带 AI 流水线与 3 MP 传感器的板型。本页覆盖模组规格、传感器、引脚表、Octal PSRAM 约束与分区规划。软件侧见 [系统架构](n16r8-architecture.md)，家族统一设计见 [ESP-Cam 总架构](espcam-architecture.md)。

## 模组规格

| 项 | 规格 |
|---|---|
| SoC | ESP32-S3，Xtensa LX7 双核 @ 240 MHz |
| Flash | 16 MB Quad SPI |
| PSRAM | **8 MB Octal**（不是 Quad——配置错了直接异常） |
| USB | USB-OTG + USB-Serial/JTAG（`/dev/ttyACM0`，115200） |
| 供电 | 5V USB，典型 ~500 mA（相机 + WiFi + AI 全开） |

## 传感器：OV3660（3 MP）

- 最大分辨率 QXGA 2048×1536，PID `0x77`，SCCB 接口，XCLK 默认 20 MHz
- 帧缓冲在 PSRAM、双缓冲（`fb_count=2`）、JPEG 直出
- 开 AI 特性时锁定 VGA 640×480（AI 缓冲固定尺寸）
- 分辨率档位 0-15（96×96 到 UXGA 1600×1200），以 `GET /api/camera` 的 `supported_resolutions` 下发为准

## 引脚表（GOOUUU 板）

| 信号 | GPIO | 信号 | GPIO |
|---|---|---|---|
| XCLK | 15 | D7 | 16 |
| SIOD (SDA) | 4 | D6 | 17 |
| SIOC (SCL) | 5 | D5 | 18 |
| D0 | 11 | VSYNC | 6 |
| D1 | 9 | HREF | 7 |
| D2 | 8 | PCLK | 13 |
| D3 | 10 | PWDN/RESET | 未接（-1） |
| D4 | 12 | | |

> 这是 GOOUUU 板的映射。不同厂家的 N16R8 板引脚可能不同，以本仓 `camera_driver.c` 为准，**不要从其他板复制引脚表**。

补光灯 LED：GPIO 2/3/46 启动时探测。

## Octal PSRAM 强制约束

```ini
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MODE_OCT=y              # R8 模组是 Octal，不是 Quad
CONFIG_SPIRAM_BOOT_INIT=y
CONFIG_SPIRAM_USE_MALLOC=y
CONFIG_SPIRAM_MALLOC_ALWAYSINTERNAL=16384
CONFIG_SPIRAM_MALLOC_RESERVE_INTERNAL=32768
CONFIG_ESP32S3_DATA_CACHE_LINE_64B=y  # 64B cache line 对 Octal DDR 是强制的
```

最后一条最容易漏：32B cache line 在 Octal DDR 模式下造成**静默数据损坏**，不是编译错误。帧缓冲与 AI 灰度缓冲都必须在 PSRAM。

## 分区规划（16 MB）

| 分区 | 偏移 | 大小 | 用途 |
|---|---|---|---|
| nvs | 0x9000 | 24 KB | 配置（逐键存储） |
| phy_init | 0xf000 | 4 KB | PHY 校准数据 |
| ota_0 | 0x10000 | 5 MB | 固件槽 A |
| ota_1 | 0x510000 | 5 MB | 固件槽 B |
| otadata | 0xa10000 | 8 KB | OTA 槽选择 |
| spiffs | 0xa12000 | 512 KB | Web UI 资产 |

双 OTA 槽已就绪；Web OTA 端点开发中（`/api/capabilities` 的 `ota` 当前为 `false`）。

## 串口与权限

Linux 下设备为 `/dev/ttyACM0`（USB-Serial/JTAG，开串口不复位板子）。用户需在 `uucp`（Arch）或 `dialout`（Debian/Ubuntu）组。找端口：

```bash
ls /dev/serial/by-id/
```

## 散热

连续推流 + AI 检测时传感器与 SoC 都会发热，长期部署保证通风；`/api/status` 的 `chip_temp` 可监控。
