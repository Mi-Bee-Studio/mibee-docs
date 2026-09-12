# N16R8 Web API

This board fully implements the family contract (v1.2) — core endpoints, envelope, auth and field naming are documented in [Unified API design](espcam-api.md). This page covers only the N16R8 capability matrix and board-specific endpoint details. Everything is on `:80`; writes take the `X-Password` header.

## Capability Matrix (`GET /api/capabilities`)

| Capability | Value | Notes |
|---|---|---|
| `ai` | `true` | Face/motion/QR pipeline (unique to this board) |
| `led` / `flash_led` | `true` | Flash LED PWM brightness 0-100 |
| `onvif` / `rtsp` / `mdns` | `true` | ONVIF SOAP + WS-Discovery, RTSP digest |
| `sd` / `audio` / `mic` / `websocket` | `false` | No SD slot, no audio, no WS push |
| `ota` | `false` (in development) | Dual OTA partitions reserved; web endpoint not registered |

`api_version` returns the current contract version string.

## Board-Specific Endpoints

### AI toggles: `POST /api/ai`

```json
{ "face": true, "motion": false, "qr": true }
```

Applied **live** (`ai_enable()` drives the pipeline directly, not just the config) and persisted. The response echoes the current state of all three.

### AI results: `GET /api/ai/status`

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

Returns 404 when the AI pipeline is not running. The frontend polls at 500ms (only while AI is enabled); `seq` drops stale frames. Note that `ai_status` reflects the live pipeline state, not config values.

### Flash LED: `POST /api/led` / `GET /api/led`

`{"brightness":0-100}`; GET reads the current brightness back.

### Camera control: `GET/POST /api/camera`

- Resolution scale **0-15** (96×96 → UXGA); the dynamic list from `supported_resolutions` is authoritative
- Tuning fields (-2…+2): `cam_brightness/contrast/saturation/sharpness`, `cam_hmirror`, `cam_vflip` apply live
- Resolution/quality changes trigger the coordinated reinit (see [architecture](n16r8-architecture.md))
- **AI safety check**: submitting a non-VGA framesize while any AI feature is enabled → 400 `"Disable AI to use non-VGA resolution"`

### RTSP

`rtsp://<ip>:554/stream`, **digest auth mandatory** (credentials `rtsp_user/rtsp_pass`, default admin/admin, changed via `POST /api/config`). Bad credentials get 401.

## Config Keys (`GET/POST /api/config`)

Management keys beyond the family-common set: `ai_face_enable` / `ai_motion_enable` / `ai_qr_enable` / `rtsp_user` / `rtsp_pass` / `onvif_enable`.

`POST /api/config` validates against a **whitelist**: keys that never appear in the GET response are rejected — build request bodies from the `GET /api/config` key set.

## Polling Guidance

- `/api/status`: 500ms
- `/api/ai/status`: 500ms (only while AI enabled)
- `/api/camera`, `/api/config`: on demand (user interaction)

Related: [Unified API design](espcam-api.md) · [Web UI](n16r8-web-ui.md)
