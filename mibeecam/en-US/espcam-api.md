# ESP-Cam Unified API Design (Contract v1.2)

All four boards expose **one REST contract**: identical where undifferentiated, and where they differ, differences may only surface through capability gating + dynamic metadata — never through divergent field names, scales or semantics. `GET /api/capabilities` returns the `api_version` string. This page documents contract v1.2; the machine-copy source of truth is each repo's `docs/api-contract.md` (md5-identical across the four repos).

## Envelope & Auth

All JSON endpoints share one envelope: success `{"ok":true,"data":...}` + HTTP 200; failure `{"ok":false,"error":"<msg>"}` + 400/401/404/500/503. Writes authenticate with the `X-Password` header. The family-wide default password is deployment-local (kept out of this public repo; empty and <6-char passwords are rejected). Password changes go through `POST /api/config` with `{"web_password":"..."}` (the old password is verified implicitly via `X-Password`). Password fields are masked `"****"` in GET responses; posting the mask back means "unchanged". CORS is fully open (`OPTIONS /*` → 204).

MJPEG streams live on the separate port `:81/stream`; viewer limits per board are ai-thinker 1 / n16r8 2 / luatos 2 / seeed 3, advertised via `status.stream_clients_max`.

## Core Endpoints (100% identical on all four)

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/status` | open | Device status (fields below) |
| GET | `/api/config` | open | Current config (passwords masked) |
| POST | `/api/config` | write | Partial update; WiFi changes persist to NVS, applied on reboot |
| GET | `/api/capabilities` | open | Capability matrix + `api_version` |
| GET | `/api/capture` | open | Single-frame JPEG |
| GET | `/api/scan` | open | WiFi scan `{networks:[{ssid,rssi,auth}]}`, RSSI-descending |
| POST | `/api/time` | write | Set time manually |
| POST | `/api/reset` | write | Factory reset + reboot |
| POST | `/api/reboot` | write | Reboot |
| GET | `/api/auth` | open | Validate password |
| GET | `/metrics` | open | Prometheus text |

## Capability Gating

Rule: `capabilities.X == true` ⇒ the endpoint exists with identical semantics; `== false` ⇒ not registered (404/405) and the frontend never calls it.

| Capability (endpoints) | Unified semantics | AI-Thinker | N16R8 | Luatos | Seeed |
|---|---|:-:|:-:|:-:|:-:|
| `led`: `POST/GET /api/led` | `{"brightness":0-100}` | ✅ | ✅ | — | — |
| `ai`: `POST /api/ai` + `GET /api/ai/status` | `{face,motion,qr}` toggles + results | — | ✅ | — | — |
| Recording: `POST /api/record` + `GET /api/record` | `?action=start\|stop` + status | ✅ | — | — | ✅ |
| `sd`: `/api/files` · `/api/download` · `POST /api/files/batch` · `POST /api/format` | File management | ✅ | — | — | ✅ |
| `ota`: `/api/ota/info` · `upload` · `spiffs` | Raw-binary-stream OTA | ✅ | in development (capability `false`) | — (single partition) | ✅ |
| `audio`: `GET /api/audio` | G.711 μ-law 8kHz raw stream | — | — | — | ✅ |
| `websocket`: `GET /ws` | Event push | — | — | ✅ | ✅ |
| ONVIF: `/onvif/device_service` etc. | SOAP | ✅ | ✅ | ✅ | ✅ |
| RTSP `:554/stream` | **digest auth mandatory** | — | ✅ (separate credentials) | — | ✅ (web password) |

Non-boolean extension keys: `api_version`, `wifi_scan`.

## `/api/status` Core Fields

| Field | Notes |
|---|---|
| `device_name` / `firmware_version` / `uptime` | `uptime` uses `esp_timer` (immune to SNTP jumps) |
| `wifi_state` | lowercase enum `ap\|connecting\|connected\|disconnected` |
| `ip` / `wifi_rssi` / `wifi_channel` / `current_ssid` | `current_ssid` = the **actually connected** SSID |
| `wifi_net` | active slot `primary\|secondary` (dual-WiFi boards only) |
| `camera` / `resolution` | measured sensor model — **trust the device over the docs** |
| `free_heap` / `min_heap` / `free_psram` | boards without PSRAM **omit** `free_psram` |
| `stream_clients` / `stream_clients_max` | MJPEG viewers vs. limit |
| `chip_temp` | boards with a temperature sensor |
| `sd_present` / `sd_total_bytes` / `sd_free_bytes` / `sd_free_percent` / `recording` | SD-capable boards |

"Omit when not applicable" is the general rule — the frontend hides controls for missing fields. Board extension fields may be appended.

## Camera Control & Resolution Scales

`GET /api/camera` returns `resolution`, `cam_framesize`, `cam_quality`, `supported_resolutions:[{label,value}]` plus whichever tuning fields the board supports (`cam_brightness/contrast/saturation/sharpness`, `cam_hmirror`, `cam_vflip`, `day_night_mode`).

**The `value` scale is board-specific** (ai 0-3 / seeed 0-7 / n16r8 0-15 / luatos 0-3). Frontends must never hardcode a resolution table — populate from `supported_resolutions` and POST only values from that list. On n16r8, AI features lock the resolution to VGA (AI buffers are fixed 640×480), enforced in both directions.

## WebSocket Events (`/ws`)

Format: `{"type":"<event>","timestamp":<unix_s>,"data":{...}}` — `motion_started`/`motion_cleared` (with `score` 0-100), `recording_started`/`recording_stopped`, `wifi_state_changed`, plus board extensions (`health_warning`, `upload_success/failed`, `wifi_switched_ssid`…).

## SD File Management (`sd` boards)

- `GET /api/files?type=all|photos|recordings&offset=&limit=` — paginated (limit ≤ 200), response includes `total`
- `DELETE /api/files?name=&type=photo|recording`
- `POST /api/files/batch` — `{"names":[...]}` or `{"scope":"all|photos|recordings"}`; skips the currently-recording segment (counted failed); rejects `..` paths
- `POST /api/format` — seeed formats at runtime; ai-thinker reboots and formats at boot (shared camera/SD bus makes runtime formatting deadlock)

## OTA (`ota` boards)

Consumes a **raw binary stream**, not multipart:

```bash
curl -X POST http://<ip>/api/ota/upload -H 'X-Password: <pwd>' \
     -H 'Content-Type: application/octet-stream' \
     --data-binary @build/mibee_cam.bin
```

The image must fit the OTA slot; a failed SPIFFS upload leaves the partition erased (serial rescue only). Verify via `/api/ota/info` (`running_partition` flip).

## Contract Governance

v1.1 unified the default password and converged legacy divergences; v1.2 unified SD batch management and format semantics. Breaking changes must bump `api_version` and record migration notes in `docs/api-contract.md` — and that file stays md5-identical across all four repos.

Related: [Unified frontend design](espcam-webui.md) · [Unified architecture](espcam-architecture.md)
