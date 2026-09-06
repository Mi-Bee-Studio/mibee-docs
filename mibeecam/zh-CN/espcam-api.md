# ESP-Cam 统一 API 设计（契约 v1.3）

四块主板暴露**同一份 REST 契约**：无差异部分完全一致；有差异部分只允许通过"能力门控 + 动态元数据"产生，禁止字段名、数值刻度或语义分叉。`GET /api/capabilities` 的 `api_version` 即契约版本。本文是契约 v1.3 的完整说明；修改任何一块板的 API 前，先改契约源文件（各仓 `docs/api-contract.md`，四仓 md5 一致）。

## 信封与鉴权

所有 JSON 端点用统一信封：

- 成功：`{"ok":true,"data":...}` + HTTP 200
- 失败：`{"ok":false,"error":"<消息>"}` + 400/401/404/500/503

写操作鉴权用 `X-Password` 请求头。**公开固件家族统一默认密码 `mibeecam2026`**（服务端拒绝空密码与少于 6 位的密码）；修改密码走 `POST /api/config` 携带 `{"web_password":"..."}`，旧密码经 `X-Password` 隐式验证。密码字段在 GET 响应中掩码为 `"****"`，POST 回传掩码值视为"未修改"。CORS 全开（`OPTIONS /*` → 204）。

MJPEG 流在独立端口 `:81/stream`，客户端上限按板为 ai-thinker 1 / n16r8 2 / luatos 2 / seeed 3，通过 `status.stream_clients_max` 下发。

## 核心端点（四板 100% 一致）

| Method | Path | Auth | 说明 |
|---|---|---|---|
| GET | `/api/status` | open | 设备状态（字段见下节） |
| GET | `/api/config` | open | 当前配置（密码掩码） |
| POST | `/api/config` | write | 部分更新；WiFi 变更写 NVS、重启生效 |
| GET | `/api/capabilities` | open | 能力矩阵 + `api_version` |
| GET | `/api/capture` | open | 单帧 JPEG |
| GET | `/api/scan` | open | WiFi 扫描 `{networks:[{ssid,rssi,auth}]}`，RSSI 降序 |
| POST | `/api/time` | write | 手动设时间 `{year,month,day,hour,min,sec}`（v1.3 补齐 ai-thinker） |
| POST | `/api/reset` | write | 恢复出厂并重启 |
| POST | `/api/reboot` | write | 重启 |
| GET | `/api/auth` | open | 校验密码 `{auth,password_set}` |
| GET | `/metrics` | open | Prometheus 文本 |

## 能力门控

规则：`capabilities.X == true` ⇒ 对应端点必须存在且语义一致；`== false` ⇒ 端点不注册（404/405），前端保证永不调用。

**能力值语义（v1.3 条款）**：布尔能力在单次运行的设备上必须恒定（编译期常量或 Kconfig 条件）；唯一例外是硬件在场性探测（seeed 的 `sd` 随插拔变化）。`api_version` 必须与契约一致，禁止漂移。

| 能力（端点） | 统一语义 | AI-Thinker | N16R8 | Luatos | Seeed |
|---|---|:-:|:-:|:-:|:-:|
| `led`：`POST/GET /api/led` | `{"brightness":0-100}`；简单 GPIO 板 0=灭 | ✅ | ✅ | — | — |
| `ai`：`POST /api/ai` + `GET /api/ai/status` | `{face,motion,qr}` 开关与结果 | — | ✅ | — | — |
| 录像：`POST /api/record` + `GET /api/record` | `?action=start\|stop` 与状态 | ✅ | — | — | ✅ |
| `sd`：`/api/files` · `/api/download` · `POST /api/files/batch` · `POST /api/format` · `GET /api/storage` | 文件与存储管理（见下节） | ✅ | — | — | ✅ |
| `ota`：`/api/ota/info` · `upload` · `spiffs` · **`POST /api/ota {"url":...}`** | 裸二进制流 OTA + URL 触发（v1.3 四板统一，仅 http://） | ✅ | ✅ | —（单分区无 OTA） | ✅ |
| `audio`：`GET /api/audio` | G.711 μ-law 8 kHz 裸流 | — | — | — | ✅ |
| `websocket`：`GET /ws` | 事件推送（见下节） | — | — | ✅ | ✅ |
| ONVIF：`/onvif/device_service` 等 | SOAP（config `onvif_enable` 可关） | ✅ | ✅ | ✅ | ✅ |
| RTSP `:554/stream` | **必须 digest 鉴权**（config `rtsp_user`/`rtsp_pass`，v1.3 起两板同源） | — | ✅ | — | ✅ |

非布尔扩展键：`api_version`、`wifi_scan`。

## `/api/status` 核心字段

| 字段 | 说明 |
|---|---|
| `device_name` / `firmware_version` / `uptime` | 基本信息（uptime 用 `esp_timer`，不受 SNTP 影响） |
| `wifi_state` | 小写枚举 `ap\|connecting\|connected\|disconnected` |
| `ip` / `wifi_rssi` / `wifi_channel` / `current_ssid` | `current_ssid` 是**实际连接**的 SSID（区别于配置值） |
| `wifi_net` | 当前配置槽位 `primary\|secondary`（仅双 WiFi 板） |
| `camera` / `resolution` | 实测传感器型号与当前分辨率——**信设备不信文档** |
| `free_heap` / `min_heap` / `free_psram` | 无 PSRAM 的板**省略** `free_psram` 字段 |
| `stream_clients` / `stream_clients_max` | MJPEG 观众数与上限 |
| `chip_temp` | 有温度传感器的板返回（°C） |
| `sd_present` / `sd_total_bytes` / `sd_free_bytes` / `sd_free_percent` / `recording` | SD 能力板的存储状态 |

"不适用即省略"是通用规则：不支持的字段直接不出现在 JSON 里，前端按字段缺省隐藏控件。板级扩展字段允许追加。

## 相机控制与分辨率刻度（v1.3 统一）

`GET /api/camera` 返回 `resolution`、`cam_framesize`、`cam_quality`、`supported_resolutions:[{label,value}]`、`quality_min`/`quality_max`（滑杆边界 10-63）、`res_cap_source`（上限被 sensor/board/memory 哪层钳制）及该板支持的微调字段（`cam_brightness/contrast/saturation/sharpness`、`cam_hmirror`、`cam_vflip`、`day_night_mode`）。

**`value` 数值刻度全家族统一为 esp32-camera 组件的 `framesize_t` 枚举**（v1.3 起：QVGA=6, VGA=10, SVGA=11, XGA=12, HD=13, SXGA=14, UXGA=15；四仓组件 md5 一致）。旧板自有刻度（ai 0-3 / seeed 0-5 / luatos 0-3）在固件升级时由各仓 config 迁移函数自动翻译 NVS 存量值，无需人工干预。前端禁止硬编码分辨率表，只从 `supported_resolutions` 填充下拉框，POST 只回传列表内的 value（越界一律 400）。分辨率上限是三层交集 `min(传感器, 板级实测, 运行时 fb 预算)`。

## 配置契约（v1.3 起独立成文）

配置子系统（持久化格式、字段名与取值域、校验矩阵、默认值、迁移策略、SD 卡 provisioning 格式）由 `docs/config-contract.md` v1.0 统一：全家族逐键 NVS（`mibee_cfg` 命名空间 + `schema_ver` 版本键），`cam_framesize` 用上节统一刻度，motion 为超集模型（enabled/sensitivity 0-100/cooldown_s/active_interval_s），timelapse 为 8 字段动态模型（静态/动态、min/max 衰减）。HTTP `/api/config` 的字段名以契约 §3 为准，密码类掩码，`GET` 含 `"schema_version"`。

## WebSocket 事件（`/ws`，websocket 能力板）

统一格式 `{"type":"<event>","timestamp":<unix_s>,"data":{...}}`：

- `motion_started` / `motion_cleared`：移动侦测翻转，data 含 `score` 0-100
- `recording_started` / `recording_stopped`：录像启停
- `wifi_state_changed`：`{"state":"connected|..."}`
- 板级扩展：`health_warning`、`upload_success/failed`、`wifi_switched_ssid` 等

## SD 文件管理（`sd` 能力板）

- `GET /api/files?type=all|photos|recordings&offset=&limit=`：分页（limit ≤ 200），响应含 `total`
- `DELETE /api/files?name=&type=photo|recording`：`type` 缺省 photo
- `POST /api/files/batch`：`{"names":[...]}` 或 `{"scope":"all|photos|recordings"}`；跳过正在写入的录像段（计 failed）；拒绝含 `..` 的路径
- `POST /api/format`：seeed 运行时格式化；ai-thinker 走"申请 → 重启 → 开机格式化"（GPIO14 相机/SD 共享总线，运行中格式化必挂死）
- `GET /api/storage`（v1.3 收编）：存储详情 `total_mb/free_mb/usage_pct/mounted`

## OTA（`ota` 能力板）

吃**裸二进制流**，不是 multipart：

```bash
curl -X POST http://<ip>/api/ota/upload -H 'X-Password: <pwd>' \
     -H 'Content-Type: application/octet-stream' \
     --data-binary @build/mibee_cam.bin      # 固件 → 备用槽 → 自动重启
curl -X POST http://<ip>/api/ota/spiffs -H 'X-Password: <pwd>' \
     --data-binary @build/spiffs.bin         # UI → 整擦 SPIFFS → 自动重启
curl -X POST http://<ip>/api/ota -H 'X-Password: <pwd>' \
     -H 'Content-Type: application/json' \
     -d '{"url":"http://192.168.1.10:8000/mibee_cam.bin"}'   # v1.3：URL 触发（仅 http://）
```

镜像必须 ≤ OTA 槽尺寸；上传中途失败 SPIFFS 即丢（只能串口救）。成功后用 `/api/ota/info` 看 `running_partition` 切换验证。Luatos 为单 factory 分区，按设计无 OTA，始终串口烧录。

## 串口 AT 控制台（契约 `docs/at-command.md` v1.1）

四板共用同一份 AT 核心实现（`main/at_command.c` md5 一致 + 板级 `main/at_port.c`）：核心指令 `AT / AT+HELP / AT+GMR / AT+WIFI?|= / AT+WIFISCAN / AT+IP? / AT+STATUS / AT+CAMRES?|= / AT+CAMQUAL?|= / AT+REBOOT / AT+RESTORE`，可选 `AT+CFGGET= / AT+CFGSET= / AT+SAVE`（白名单读写，密码类只写不读）。响应框架：数据行 `+NAME:内容`、成功 `OK`、失败 `ERROR: 原因`；输入大小写不敏感；分辨率 value 同 HTTP 统一刻度。通道：ai-thinker/n16r8（CH340 UART0）、luatos（CH343 UART0，开串口即复位——PIT-003）、seeed（USB-JTAG CDC）。

## 契约治理

- 版本演进：v1.1 统一默认密码与遗留差异收敛；v1.2 统一 SD 批量管理与格式化语义；**v1.3 分辨率刻度统一 framesize_t、配置契约独立成文（config-contract v1.0）、能力值语义条款、OTA URL 触发四板统一、`GET /api/storage` 收编、`POST /api/time` 补齐**。破坏性变更必须 bump `api_version` 并在 `docs/api-contract.md` 记录迁移说明。
- 任何板的 API/配置/AT 改动先改对应契约文档，四仓同步（三份契约文档四仓 md5 一致是 CI 前的人工检查项）。

相关阅读：[统一前端设计](espcam-webui.md) · [总架构](espcam-architecture.md)
