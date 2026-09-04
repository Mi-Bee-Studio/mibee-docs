# ESP-Cam 统一 API 设计（契约 v1.2）

四块主板暴露**同一份 REST 契约**：无差异部分完全一致；有差异部分只允许通过"能力门控 + 动态元数据"产生，禁止字段名、数值刻度或语义分叉。`GET /api/capabilities` 的 `api_version` 即契约版本。本文是契约 v1.2 的完整说明；修改任何一块板的 API 前，先改这里（源文件为各仓 `docs/api-contract.md`，四仓 md5 一致）。

## 信封与鉴权

所有 JSON 端点用统一信封：

- 成功：`{"ok":true,"data":...}` + HTTP 200
- 失败：`{"ok":false,"error":"<消息>"}` + 400/401/404/500/503

写操作鉴权用 `X-Password` 请求头。家族统一默认密码 `mibeecam2026`（公开初始密码，完成首次配置后请立即修改；服务端拒绝空密码与少于 6 位的密码）；修改密码走 `POST /api/config` 携带 `{"web_password":"..."}`，旧密码经 `X-Password` 隐式验证。密码字段在 GET 响应中掩码为 `"****"`，POST 回传掩码值视为"未修改"。CORS 全开（`OPTIONS /*` → 204）。

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
| POST | `/api/time` | write | 手动设时间 `{year,month,day,hour,min,sec}` |
| POST | `/api/reset` | write | 恢复出厂并重启 |
| POST | `/api/reboot` | write | 重启 |
| GET | `/api/auth` | open | 校验密码 `{auth,password_set}` |
| GET | `/metrics` | open | Prometheus 文本 |

## 能力门控

规则：`capabilities.X == true` ⇒ 对应端点必须存在且语义一致；`== false` ⇒ 端点不注册（404/405），前端保证永不调用。

| 能力（端点） | 统一语义 | AI-Thinker | N16R8 | Luatos | Seeed |
|---|---|:-:|:-:|:-:|:-:|
| `led`：`POST/GET /api/led` | `{"brightness":0-100}`；简单 GPIO 板 0=灭 | ✅ | ✅ | — | — |
| `ai`：`POST /api/ai` + `GET /api/ai/status` | `{face,motion,qr}` 开关与结果 | — | ✅ | — | — |
| 录像：`POST /api/record` + `GET /api/record` | `?action=start\|stop` 与状态 | ✅ | — | — | ✅ |
| `sd`：`/api/files` · `/api/download` · `POST /api/files/batch` · `POST /api/format` | 文件管理（见下节） | ✅ | — | — | ✅ |
| `ota`：`/api/ota/info` · `upload` · `spiffs` | 裸二进制流 OTA | ✅ | 开发中（分区已备，能力位 `false`） | —（单分区无 OTA） | ✅ |
| `audio`：`GET /api/audio` | G.711 μ-law 8 kHz 裸流 | — | — | — | ✅ |
| `websocket`：`GET /ws` | 事件推送（见下节） | — | — | ✅ | ✅ |
| ONVIF：`/onvif/device_service` 等 | SOAP | ✅ | ✅ | ✅ | ✅ |
| RTSP `:554/stream` | **必须 digest 鉴权** | — | ✅（独立账号） | — | ✅（web 密码） |

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

## 相机控制与分辨率刻度

`GET /api/camera` 返回 `resolution`、`cam_framesize`、`cam_quality`、`supported_resolutions:[{label,value}]` 及该板支持的微调字段（`cam_brightness/contrast/saturation/sharpness`、`cam_hmirror`、`cam_vflip`、`day_night_mode`）。

**`value` 数值刻度是板相关的**（ai 0-3 / seeed 0-7 / n16r8 0-15 / luatos 0-3）。前端禁止硬编码分辨率表，只从 `supported_resolutions` 填充下拉框，POST 只回传列表内的 value。n16r8 上 AI 与 VGA 联动（AI 缓冲固定 640×480，开 AI 即锁 VGA），前端双向强制。

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

## OTA（`ota` 能力板）

吃**裸二进制流**，不是 multipart：

```bash
curl -X POST http://<ip>/api/ota/upload -H 'X-Password: <pwd>' \
     -H 'Content-Type: application/octet-stream' \
     --data-binary @build/mibee_cam.bin      # 固件 → 备用槽 → 自动重启
curl -X POST http://<ip>/api/ota/spiffs -H 'X-Password: <pwd>' \
     --data-binary @build/spiffs.bin         # UI → 整擦 SPIFFS → 自动重启
```

镜像必须 ≤ OTA 槽尺寸；上传中途失败 SPIFFS 即丢（只能串口救）。成功后用 `/api/ota/info` 看 `running_partition` 切换验证。

## 契约治理

- 版本演进：v1.1 统一默认密码与遗留差异收敛；v1.2 统一 SD 批量管理与格式化语义。破坏性变更必须 bump `api_version` 并在 `docs/api-contract.md` 记录迁移说明。
- 任何板的 API 改动先改契约文档，四仓同步（该文档四仓 md5 一致是 CI 前的人工检查项）。

相关阅读：[统一前端设计](espcam-webui.md) · [总架构](espcam-architecture.md)
