# ESP-Cam 开发知识库

四个仓库踩过的坑，按"症状 → 真因 → 修复"整理。这些条目来自真实排障记录（内部维护于各仓 `AGENTS.md` 与跨仓坑库，本页是公开脱敏版）。调试任何"WiFi 不稳定 / 设备反复重启 / 网页打不开"的报告前，先按关键词对号入座，再怀疑硬件。

## 网络与稳定性

### EMFILE 重启循环——"WiFi 不稳定"的头号真凶

**症状**：页面时好时坏、HTTP 被重置、ping 抖动大，看起来像射频问题。

**真因链**：浏览器端 MJPEG 断线自愈每约 7 秒重连一次 → 每个被踢连接在设备侧留 TIME_WAIT（lwIP 默认 2×MSL = 120 秒）→ 默认 `LWIP_MAX_SOCKETS=10` 被打爆 → httpd `accept` 报 EMFILE → 健康探测连续失败触发自愈重启 → 重启后十几秒再爆，循环。

**日志签名**（三连即可确诊）：

```
E httpd: httpd_accept_conn: error in accept (23)
W health_monitor: httpd :80 probe failed (n/6)
rst:0xc (SW_CPU_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
```

**修复**：`sdkconfig.defaults` 加 `CONFIG_LWIP_MAX_SOCKETS=16`（seeed 24）+ `CONFIG_LWIP_TCP_MSL=15000`（TIME_WAIT 缩到 30 秒）。改 defaults 后必须删掉生成的 sdkconfig 重配，否则新值静默不生效。四仓均已合入。

**教训**：见 `accept (23)` 先查 socket 表，别先怀疑天线；重启循环期间测得的 ping 抖动没有诊断价值。

### 自愈误杀：探测失败 ≠ 设备死了

任何"探测失败 → 重启"的自愈逻辑，重启前必须排除两类误杀：**网络不可达**（WiFi 掉线时本地探测必失败）和 **EMFILE**（探测自身要 socket，表满时必失败）。规则：WiFi 未连接时不计数；EMFILE 场景修根因而不是放宽重启阈值。

### "板子很慢"先查连的哪个 AP

**症状**：某块板网页秒开要 3 秒、ping 平均几百毫秒，同一时刻另一块板同网推流 ping 仅 6 ms。

**真因（三层叠加）**：① 连到了 40 MHz（HT40）的 2.4G AP——占双信道、灵敏度差约 6 dB，弱信号下丢包恶化；② AMPDU 聚合被关（早年绕驱动 stall 的权宜），吞吐塌到 1-3 Mbps，MJPEG 吃满空中时间，ping/HTTP 全排队；③ 关联瞬间的 RSSI 不可信（关联时 -56、稳态 -70）。

**修复**：STA 强制 20 MHz 带宽 + 重开 AMPDU（S3 板现行配置）。ESP32 日志的 `connected with <ssid>, channel N, 40D|BW20` 一行就能读出全部链路参数；多板同网对比延迟是定位利器——一快多慢时先查共性（连的 AP、信道/带宽、聚合），比改固件见效快。

### S3 板高温是独立故障源

芯片温度持续高于规格上限（实测日志 60 秒周期报 93 °C 级别告警）时，一切延迟问题都被放大。排障先看 `/api/status` 的 `chip_temp` 再下结论；高温本身是散热/负载问题，固件不可根治，但连续推流 + 高温会显著恶化网络表现。

## 内存与并发

### 单 framebuffer 板的通用约束（无 PSRAM 板）

- MJPEG 双客户端会把堆压到百字节级 → 流任务自断。解法是硬上限单观众（新连接 LRU 踢旧观众）+ 堆水位纵深防御。
- **开机初期的高堆瞬时值会绕过堆水位门槛**——门槛只能当纵深，不能当唯一闸门。
- 摄像头热重配（deinit + init）与并发取帧是致命竞态：无 PSRAM 板改"保存 → 应答 → 1 秒后重启应用"。有 PSRAM（fb_count=2）的板可在线协调，**板间不通用，别互抄**。

### 事件/定时类的隐性编译期上限

功能叠加会悄悄越过编译期上限且部分静默失效：事件总线订阅表、httpd `max_uri_handlers`、lwIP socket 表都是同一类坑。加订阅方之前先数表容量。**通配路由先注册会遮蔽精确端点**（`GET /*` 导致 `/api/camera` 运行时 404）——注册顺序：精确端点 → `/ws` → 静态兜底。

### uptime 用 esp_timer，别用 time(NULL)

SNTP 同步后 `time(NULL)` 从 epoch 起跳（日志会打出天文数字级 Uptime）。一切时长统计用 `esp_timer_get_time()`。

## 构建与固件工程

### 改 sdkconfig.defaults 不生效

`sdkconfig` 是生成物；改了 defaults 不删 sdkconfig，新配置**静默不生效**。铁律：改 defaults → 删 sdkconfig → 重新 set-target → build → 烧录后 `grep` 生成物反查确认。

### 前端与固件构建耦合

`main/web_ui/` 资产构建期打包进 SPIFFS——改 HTML/JS/CSS 必须重新构建烧录；CMake 的 spiffs 镜像要声明显式文件级 `DEPENDS`（目录级依赖不随内容编辑触发）。

### 禁止手改 managed_components

上游组件的改动在 `fullclean`/新克隆即丢、CI 与本地静默分叉。正道：`patches/` + 根 CMake 拷贝步骤，或 vendor 进 `components/`。

### 共享 SPA 四文件纪律

三 S3 仓的 `index.html / app.js / i18n.js / style.css` md5 必须一致：改任何一仓，同步其余两仓并校验。功能差异用能力探测降级，绝不按板分叉（详见[统一前端设计](espcam-webui.md)）。

## 硬件与平台

### 信设备不信文档

文档写 OV2640、板子实戴 OV5640 的出厂批次真实存在（驱动自动识别）。传感器/配置一律以 boot log 或 `/api/status` 的 `camera` 字段为准；发现矛盾上报修正文档，不要按文档"纠正"设备。

### ai-thinker（原版 ESP32）专属

- 相机必须在 STA 连上后初始化（先开相机触发 DMA freeze）
- GPIO14 是 SD/相机共享总线：运行中格式化 SD 必挂死 → 走"重启后开机格式化"
- GPIO0 是 XCLK，不能做按键
- 每次烧录后 PHY 全校准，异常时擦 phy_init 分区
- 烧录后 RTS 卡复位：`esptool --no-stub run` 释放

### n16r8（Octal PSRAM）专属

- 8 MB **Octal** PSRAM（不是 Quad）：`CONFIG_SPIRAM_MODE_OCT=y`
- 64B cache line 对 Octal DDR 模式是强制的，32B 会静默数据损坏
- esp32-camera 走 `patches/` 补丁机制

### luatos（无 PSRAM 设计）专属

- **PSRAM 禁用是设计决策**，开了直接 boot loop
- 钉在 ESP-IDF v5.5.4（其余三仓 v6.0.1）
- 单分区布局：没有 OTA 能力，升级走串口
- WPA3 关闭、STA 强制 HT20、单观众硬上限（见上文单 framebuffer 约束）

## 对接与运维

### Web OTA 吃裸二进制流

OTA 端点不是 multipart 表单，是 `Content-Type: application/octet-stream` 的裸流（curl 示例见[统一 API](espcam-api.md)）。镜像必须 ≤ OTA 槽尺寸；SPIFFS 上传中途失败只能串口救；验证以 `running_partition` 切换 + 设备 `/app.js` md5 与仓库一致为准。

**日常升级优先走 Web OTA**（有 OTA 能力的板）：每次改动都经由 OTA 交付，固件与前端的上传链路才能持续保持可用；USB 刷机仅用于全新芯片、救砖或设备不可达。luatos 单分区无 OTA（串口是唯一途径）；n16r8 的 OTA 端点在开发中，合入前仍走 USB。

### 设备序列号的稳定来源

序列号/UUID 一律从出厂 eFuse MAC 派生（读一次缓存）。曾经各消费点在请求时实时读 WiFi MAC——WiFi 未就绪时返回全零，污染 NVR 侧的 stable_id 记账。

## 延伸阅读

各仓的深度性能与踩坑长文：[ESP32-CAM / ESP-IDF 开发中绕不开的那些坑](aicam-esp32-cam-performance.md)、[AI-Thinker 性能说明](aicam-performance.md)。接线与逐板排障见各板手册的故障排除页。
