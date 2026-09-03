# N16R8 开发指南

从零构建、烧录与二次开发本仓的实操手册。家族级约定（四仓不互抄引脚/分区/sdkconfig、`managed_components/` 红线等）见 [开发知识库](espcam-kb.md)。

## 环境要求

| 工具 | 版本 | 说明 |
|---|---|---|
| ESP-IDF | **v6.0.1（钉死）** | 不要用其他版本 |
| Python | 3.14+ | 构建系统依赖 |
| 串口 | `/dev/ttyACM0` | USB-Serial/JTAG，115200；用户需在 `uucp`/`dialout` 组 |

## 构建与烧录

```bash
git clone https://github.com/Mi-Bee-Studio/esp32s3-n16r8-cam.git
cd esp32s3-n16r8-cam
source ~/.espressif/v6.0.1/esp-idf/export.sh   # 每个新 shell
idf.py set-target esp32s3                       # 首次必须；跳过会静默构建失败
idf.py build
idf.py -p /dev/ttyACM0 flash monitor
```

产物：`build/mibee_cam.bin`（固件）、`build/bootloader/bootloader.bin`、`build/partition_table/partition-table.bin`、`build/ota_data_initial.bin`、`build/spiffs.bin`（Web UI）。

## esp32-camera 补丁机制

本仓对上游 `espressif/esp32-camera` 的修改**不进 `managed_components/`**（该目录是生成物，`fullclean` 即丢、CI 与本地分叉）。正道是 `patches/` + 根 CMake 拷贝步骤：

1. 修改放在 `patches/<组件>/` 下的补丁文件
2. 根 `CMakeLists.txt` 在配置阶段把补丁后的文件拷贝覆盖 `managed_components/` 对应文件
3. `fullclean` 或新克隆后重新构建会自动重新应用

led_strip 组件的 IDF v6.0.1 编译修复就是走这个机制的实例。

## 改 sdkconfig.defaults 的纪律

`sdkconfig` 是生成物。改了 `defaults` 之后：

```bash
rm sdkconfig
idf.py set-target esp32s3
idf.py build
grep CONFIG_LWIP_MAX_SOCKETS sdkconfig   # 反查确认新值生效
```

不删 `sdkconfig` 直接 build，新值**静默不生效**（家族级坑，烧出去的还是旧配置）。

## 改 Web UI 的纪律

`main/web_ui/` 四文件在构建期打包进 SPIFFS：

- 改 HTML/JS/CSS 后必须 `idf.py build` + flash 才生效（浏览器强刷无效）
- 根 CMake 的 spiffs 镜像已声明显式文件级 `DEPENDS`，改文件内容会触发重打包
- **四文件与 luatos/seeed 共享：改任何一仓，同步其余两仓并 md5 校验**，能力差异用运行时探测，禁止按板分叉

## 配置键参考（NVS 命名空间 `mibee_cfg`）

逐键存储（u8/i8 类型，FreeRTOS 互斥锁保护）。HTTP 层的键名与语义见 [Web API](n16r8-web-api.md)；改配置结构体的第一手参考是 `config_manager.c` 的键表。

## AT 指令

UART0 上的串口配置通道（115200），用于无网络时的最小配置：WiFi 设置、状态查询。指令集见 `at_command.c` 或[ Luatos 板的 AT 手册](luatos-at-commands.md)（两仓指令族同源，本板为子集）。

## 常用排障入口

- 启动日志按编号步骤输出，卡在哪一步一目了然
- `/api/status`：`camera` 字段是**实测**传感器型号（文档矛盾时信它）
- "WiFi 不稳定/反复重启"：先查[知识库 EMFILE 条目](espcam-kb.md)的日志签名，再查射频
- 烧录后板子无串口输出：`esptool --no-stub run` 释放 RTS 复位线（CH340 家族通病）

相关阅读：[硬件手册](n16r8-hardware.md) · [系统架构](n16r8-architecture.md)
