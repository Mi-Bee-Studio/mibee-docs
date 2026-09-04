# N16R8 Web API

本板完整实现家族统一契约（v1.2）——核心端点、信封、鉴权与字段命名见 [统一 API 设计](espcam-api.md)，本页只讲 N16R8 的能力矩阵与板级端点细节。所有端点在 `:80`，写操作带 `X-Password` 头。

## 能力矩阵（`GET /api/capabilities`）

| 能力位 | 值 | 说明 |
|---|---|---|
| `ai` | `true` | 人脸/移动/QR 流水线（本板独有） |
| `led` / `flash_led` | `true` | 补光灯 PWM 亮度 0-100 |
| `onvif` / `rtsp` / `mdns` | `true` | ONVIF SOAP + WS-Discovery、RTSP digest |
| `sd` / `audio` / `mic` / `websocket` | `false` | 无 SD 卡槽、无音频、无 WS 推送 |
| `ota` | `false`（开发中） | 双 OTA 分区已备，Web 端点未注册 |

`api_version` 返回当前契约版本字符串。

## 板级端点详解

### AI 开关：`POST /api/ai`

```json
{ "face": true, "motion": false, "qr": true }
```

**即时生效**（`ai_enable()` 直通流水线，不只是写配置）并持久化。响应回读当前三项状态。

### AI 结果：`GET /api/ai/status`

```json
{
  "ok": true,
  "data": {
    "face": { "count": 2, "boxes": [
      { "x": 100, "y": 150, "w": 80, "h": 100, "confidence": 0.95 },
      { "x": 300, "y": 200, "w": 70, "h": 90,  "confidence": 0.88 } ] },
    "motion": { "score": 23 },
    "qr":     { "count": 1, "codes": ["https://example.com"] },
    "seq": 12345
  }
}
```

AI 流水线未运行时 404。前端 500 ms 轮询（仅 AI 开启时），`seq` 用于丢弃过期帧。注意 `ai_status` 反映的是流水线实时状态，不是配置值。

### 补光灯：`POST /api/led` / `GET /api/led`

`{"brightness":0-100}`；读回当前亮度。

### 相机控制：`GET/POST /api/camera`

- 分辨率刻度 **0-15**（96×96 → UXGA），动态表以 `supported_resolutions` 为准
- 微调字段（-2…+2）：`cam_brightness/contrast/saturation/sharpness`、`cam_hmirror`、`cam_vflip` 即时生效
- 分辨率/质量变更触发协调式重配（详见 [系统架构](n16r8-architecture.md)）
- **AI 安全检查**：任何 AI 特性开启时提交非 VGA 档位 → 400 `"Disable AI to use non-VGA resolution"`

### RTSP

`rtsp://<ip>:554/stream`，**强制 digest 鉴权**（凭证 `rtsp_user/rtsp_pass`，默认 admin/admin，经 `POST /api/config` 修改）。错误凭证 401。

## 配置键（`GET/POST /api/config`）

本板在家族通用键之外的管理键：`ai_face_enable` / `ai_motion_enable` / `ai_qr_enable` / `rtsp_user` / `rtsp_pass` / `onvif_enable`。

注意 `POST /api/config` 是**白名单校验**：响应里没出现的键发送会被拒绝——前端按 `GET /api/config` 返回的键集构造请求体。

## 轮询建议

- `/api/status`：500 ms
- `/api/ai/status`：500 ms（仅 AI 开启时）
- `/api/camera`、`/api/config`：按需（用户交互时）

相关阅读：[统一 API 设计](espcam-api.md) · [Web UI](n16r8-web-ui.md)
