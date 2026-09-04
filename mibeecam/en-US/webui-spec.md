# MiBee Camera Web API Unified Specification (SPEC v1)

This specification defines the unified contract for the Web management API of MiBee camera devices. It is the single source of truth between the device implementations and the shared frontend (mibee-webui). The specification version is advertised via `spec_version` in `/api/capabilities`, currently `"1"`.

| Implementation | Deployment |
|------|------|
| `mibee-eye-raspi-rs` (Rust Raspberry Pi camera) | HTTP :8088, single camera (fixed id `"0"`) |
| `mibee-eye-raspi-go` (Go Raspberry Pi camera) | HTTP :8088, single camera (fixed id `"0"`) |
| `notebook-cam` (laptop camera) | HTTPS :8443 (TLS enforced), multi-camera CRUD |

The reference frontend is a zero-build ES Modules implementation embedded directly by the three device repositories; all frontend feature switches follow the advertised capabilities.

## 0. Conventions

- **Envelope**: all JSON endpoints (unless noted) use a unified envelope.
  - Success: `{"ok":true,"data":<payload>}`
  - Failure: `{"ok":false,"error":"<machine code>","message":"<human readable>"}` + a semantic HTTP status code
  - Binary endpoints (snapshot / MJPEG / MSE / metrics / static assets) are not enveloped.
- **Error code table** (values of the `error` field, one-to-one with HTTP status):

  | error | HTTP | Meaning |
  |-------|------|------|
  | `bad_request` | 400 | Invalid request body / parameters |
  | `unauthorized` | 401 | Not logged in / session expired / CSRF check failed |
  | `forbidden` | 403 | Logged in but not allowed |
  | `not_found` | 404 | Resource does not exist |
  | `conflict` | 409 | State conflict (stream already running, etc.) |
  | `setup_required` | 503 | First boot, admin setup not completed yet |
  | `rate_limited` | 429 | Rate limited / login lockout |
  | `not_implemented` | 501 | Advertised capability but backend not implemented |
  | `internal_error` | 500 | Internal error |

- **Versioning**: within the same `spec_version` only additive changes are allowed; breaking changes must bump the version.
- **Authentication**: session cookie + CSRF double submit (see §2). Except for the public endpoints in §1, every API requires authentication. Static assets (the frontend itself) are public.
- **Extension mechanism**: Core endpoints must be implemented by all three devices; Extension endpoints exist only on devices that implement them and **must** be advertised truthfully in `/api/capabilities`. The frontend gates every feature on capabilities and never probes the device.

## 1. Public Endpoints

| Method | Path | Description |
|------|------|------|
| GET | `/api/health` | Liveness probe. `{"ok":true,"data":{"status":"ok","uptime":<secs>}}` |
| GET | `/metrics` | Prometheus text format (metric names are device-specific, see Appendix A5) |
| GET | `/`, `/style.css`, `/js/*` | Embedded frontend static assets |

## 2. Authentication (Core)

Model: single administrator (username + password) → server-side session → `session` cookie; write operations carry a CSRF header.

| Method | Path | Request | Response data |
|------|------|------|-----------|
| GET | `/api/auth/me` | — | Logged in: `{"username":"admin","role":"admin"}`; not logged in: 401; not initialized: 503 `setup_required` |
| POST | `/api/auth/setup` | `{"username","password"}` (password ≥ 8 chars) | `{"username"}`; establishes a session (sets cookie); 400 when already initialized |
| POST | `/api/auth/login` | `{"username","password"}`; empty/missing `username` is treated as `"admin"` (single-admin password login form) | `{"username"}`; establishes a session; wrong credentials 401; rate limited / locked 429 |
| POST | `/api/auth/logout` | — | 204, clears the session |
| POST | `/api/auth/reset` | `{"old_password","new_password"}` | `{"username"}`; invalidates all existing sessions on success |

Cookie contract:

- `session=<token>`; `HttpOnly`; `Path=/`; `SameSite=Strict`; `Secure` added for TLS deployments; valid for 24h.
- `csrf-token=<token>`; **not** HttpOnly (readable by JS); `Path=/`; `SameSite=Strict`; issued together with the session on login/setup.

CSRF contract: every `POST/PUT/DELETE/PATCH` to `/api/*` (except the auth family: login/setup/logout) must carry an `X-CSRF-Token` header matching the `csrf-token` cookie, otherwise 401.

Login failure protection: at minimum per-IP rate limiting; per-username exponential backoff lockout recommended.

## 3. Device Information (Core)

| Method | Path | Response data |
|------|------|-----------|
| GET | `/api/status` | `{"device_name","model","vendor","firmware","uptime":<secs>, ...}` + device-specific status fields (e.g. `recording`, `gb28181`, `cameras_running`) |
| GET | `/api/capabilities` | See §3.1 |

### 3.1 capabilities superset schema

```json
{
  "spec_version": "1",
  "device": {"name": "...", "model": "...", "vendor": "..."},
  "auth": {"model": "session", "setup": true},
  "multi_camera": false,
  "camera_management": false,
  "camera_control": false,
  "imaging": false,
  "ai": false,
  "ptz": false,
  "hls": false,
  "recording": false,
  "devices": false,
  "mjpeg": true,
  "mse": true,
  "webrtc": false,
  "events": ["param_changed", "ai_detection"],
  "config_apply": {"default": "restart", "sections": {"imaging": "immediate"}},
  "restart": true,
  "observability": {"metrics": true, "logs": true, "requests": true}
}
```

Field semantics:

- `multi_camera`: more than one camera (frontend shows a camera list / grid view).
- `camera_management`: camera CRUD supported (§4.2).
- `camera_control`: start/stop supported (§4.3).
- `imaging` / `ai` / `ptz` / `hls` / `recording` / `devices` / `webrtc`: the corresponding Extension endpoints exist.
- `mjpeg` / `mse`: the corresponding stream endpoints exist (frontend fallback chain MSE → MJPEG → snapshot polling).
- `events`: the event vocabulary the SSE channel actually pushes (§6).
- `config_apply`: `"restart"` (takes effect after a process restart) / `"immediate"` (takes effect at once), refined per config section; sections not listed use `default`. The frontend should annotate each config section with its apply timing and offer a restart entry after a `restart` section is saved (§5.1). Optional boolean `auto` (default `false`): when `true` (Go dialect), saving a change to a `restart` section **makes the device automatically restart itself immediately** (the save response already carries `applied:"restart"`); the frontend should enter the unified restart-wait flow (notice → poll `/api/health` → auto-reload after recovery) instead of showing a manual restart entry.
- `restart`: the device supports `POST /api/system/restart` (§5.1).
- `observability`: observability capabilities (§3.2); when absent, all three are treated as `false` and the frontend hides the resource monitor and log/request views.

### 3.2 Observability (Extension: `observability`)

Real-time monitoring and observability. All rate values are computed by a built-in 2s sampler on the device (independent of the caller's polling rhythm, so repeated polls are semantically stable); no history is stored — the frontend keeps its own rolling window.

| Method | Path | Description |
|------|------|------|
| GET | `/api/metrics/summary` | Real-time system + process resource snapshot (structure below), session auth |
| GET | `/metrics` | Prometheus text format (0.0.4), **public, no auth** (scraping convention; go keeps a dedicated 9100 port as a dialect) |
| GET | `/api/logs?limit=&level=` | Most recent logs from an in-memory ring buffer (`limit` default 200, max 1000; `level` is a minimum-level filter debug/info/warn/error), session auth |
| GET | `/api/requests?limit=` | Recent Web API request trace summaries (`limit` default 100, max 500), session auth |

`/api/metrics/summary` response data:

```json
{
  "ts": 1788320000,
  "interval_ms": 2000,
  "system": {
    "cpu_percent": 23.5,
    "load_avg": [0.4, 0.35, 0.3],
    "memory": {"total": 8589934592, "used": 3200000000, "available": 5389934592},
    "disks": [{"path": "/", "total": 61080000000, "used": 12216000000, "free": 48864000000},
               {"path": "/mnt/data", "total": 240000000000, "used": 9600000000, "free": 230400000000}],
    "network": {"rx_bytes": 123456789, "tx_bytes": 9876543,
                 "rx_rate": 12000.0, "tx_rate": 800.0}
  },
  "process": {
    "cpu_percent": 12.0,
    "rss_bytes": 123456789,
    "open_fds": 42,
    "uptime": 3600,
    "io_read_bytes": 1000000, "io_write_bytes": 2000000,
    "storage_bytes": 9600000000,
    "traffic": {"http_rx_bytes": 5000, "http_tx_bytes": 900000,
                 "rtsp_tx_bytes": 100000000, "gb28181_tx_bytes": 40000000}
  }
}
```

Field semantics:

- `system.cpu_percent`: whole-machine CPU usage since the previous sampling period (0–100, including other processes).
- `system.disks`: device-relevant mount points (root partition + recording data partition, if mounted).
- `system.network`: aggregate NIC counters and rates (bytes/sec).
- `process.cpu_percent` / `rss_bytes` / `open_fds`: CPU, resident memory, and open file descriptors of the service process.
- `process.io_read_bytes` / `io_write_bytes`: cumulative process I/O bytes (rchar/wchar of Linux `/proc/<pid>/io`, files and sockets included).
- `process.storage_bytes`: disk usage of the service = actual size of the recording data directory (accumulated from the recording index; 0 when recording is disabled).
- `process.traffic`: **application-attributed** traffic counters (not exact kernel values): HTTP request bytes (middleware statistics), RTSP/RTP outgoing bytes, GB28181 outgoing bytes. Linux provides no per-process kernel network counters, so this field is device-instrumented accumulation; rates are computed by the frontend from the difference between two polls.

`/api/requests` response data: `{"entries":[{"id":"a1b2c3","method":"GET","path":"/api/status","status":200,"duration_ms":3.2,"ts":1788320000}]}`, newest first. The middleware assigns each Web API request a `request_id` (echoed in the `X-Request-Id` response header) and records method / path / status / duration; the same `request_id` appears in related `/api/logs` entries for device-side call correlation. The RTSP/ONVIF/GB28181 dedicated port faces are not traced (covered by `/metrics` counters instead).

## 4. Camera Resource (Core)

A camera is a resource, uniformly mounted under `/api/cameras`. Single-camera devices always have exactly one camera with fixed id `"0"`.

Camera document:

```json
{
  "id": "0",
  "name": "Front Door",
  "status": "online",
  "camera_type": "csi",
  "rtsp_url": "rtsp://host:8554/stream",
  "resolution": "1280x720",
  "fps": 25
}
```

`status`: `online` (capturing) / `offline` (device unplugged); devices with `camera_control` (notebook) use `running` / `stopped` / `idle` / `offline`. The frontend "running" check = `online | running`. `camera_type`: `csi` / `usb` / `rtsp`.

### 4.1 Reading and Media (Core)

| Method | Path | Description |
|------|------|------|
| GET | `/api/cameras` | `data` is an array of Cameras |
| GET | `/api/cameras/{id}` | Single Camera; 404 when missing |
| GET | `/api/cameras/{id}/snapshot` | JPEG snapshot (`image/jpeg`), auth required |
| GET | `/api/cameras/{id}/live` | MJPEG stream (`multipart/x-mixed-replace; boundary=...`), auth required; capability `mjpeg` |
| GET | `/api/cameras/{id}/stream.mse` | chunked HTTP fMP4 (`video/mp4`), server-side muxing, first chunk is the init segment, auth required; capability `mse`. Clients use `fetch` + ReadableStream to append to a MediaSource |

MSE stream details: the init segment (`ftyp`+`moov`) is sent once, then one `moof`+`mdat` per access unit; new subscribers must wait for a keyframe before starting, and the init segment must be re-sent. After a disconnect the client simply reconnects (the server streams statelessly).

### 4.2 Camera CRUD (Extension: `camera_management`)

| Method | Path | Description |
|------|------|------|
| POST | `/api/cameras` | `{"name","camera_type","config"}` → 201 + Camera |
| PUT | `/api/cameras/{id}` | Partial update `{"name"?,"config"?,"status"?}` |
| DELETE | `/api/cameras/{id}` | Delete (must stop first) |

### 4.3 Start/Stop (Extension: `camera_control`)

| Method | Path | Description |
|------|------|------|
| POST | `/api/cameras/{id}/start` | Start capture; 409 when already running |
| POST | `/api/cameras/{id}/stop` | Stop capture; idempotent |

### 4.4 Recording Control (Extension: `recording`)

| Method | Path | Description |
|------|------|------|
| GET | `/api/cameras/{id}/recording` | `{"active":bool,"storage_path"?,"segment_secs"?,"retention_days"?}` |
| POST | `/api/cameras/{id}/recording` | `{"active":bool}`; apply timing determined by `config_apply.sections.recording` |

### 4.5 Imaging Control (Extension: `imaging`)

Parameter names follow ONVIF PascalCase (`Brightness`, `AWBMode`…).

| Method | Path | Description |
|------|------|------|
| GET | `/api/cameras/{id}/imaging/params` | `{"<Param>":<value>...}` |
| GET | `/api/cameras/{id}/imaging/options` | Numeric parameters `{"min","max","step","default"}`; enum parameters `{"enums":[...]}` |
| POST | `/api/cameras/{id}/imaging/param` | `{"name","value"}`; immediate effect; broadcasts a `param_changed` event |

### 4.6 AI Detection (Extension: `ai`)

| Method | Path | Description |
|------|------|------|
| GET | `/api/detections` | `{"detections":[{"label","confidence","bbox":[x,y,w,h]}],"model","timestamp"}`; `{"enabled":false}` when not enabled |

**bbox coordinate system**: `[x, y, w, h]` are integers in **video pixels**, origin at the top-left, in the camera's native stream resolution (the stream resolution returned by `/api/cameras`, e.g. 1280×720). Not the model input resolution, and not 0..1 normalized values — devices must map model-space coordinates back to video pixel space before returning (when a model stretches a 16:9 frame into a square input, the x/y scale factors differ and the mapping must not be skipped).

### 4.7 PTZ (Extension: `ptz`, virtual or real pan-tilt)

| Method | Path | Description |
|------|------|------|
| GET | `/api/ptz/status` | `{"pan":0.5,"tilt":0.5,"zoom":1.0}` (normalized) |
| POST | `/api/ptz/move` | `{"pan"?,"tilt"?,"zoom"?}` absolute position |

### 4.8 Host Device Enumeration (Extension: `devices`)

| Method | Path | Description |
|------|------|------|
| GET | `/api/devices/video` | `[{"index","name","formats":[string]}]` |
| GET | `/api/devices/video/{index}/formats` | `[{"width","height","format","fps"}]` |
| GET | `/api/devices/audio` | `[{"name","supported_configs":[...]}]` |

## 5. Configuration (Core)

| Method | Path | Description |
|------|------|------|
| GET | `/api/config` | Full configuration document (schema differs per device), secret fields (`password` etc.) masked as `"****"` |
| PUT | `/api/config` | **Partial merge** write: submit only the subtree to change, deep-merged into the current config; a `"****"` value written back verbatim is restored server-side to the stored value. Response `{"applied":"restart"|"immediate"}` |

Frontend responsibilities: read → render editor recursively → collect changed subtree → deep merge → PUT. Apply semantics are read from `capabilities.config_apply` and shown to the user: each config section title is annotated "requires restart" / "immediate"; after saving a change to a `restart` section, if the device advertises the `restart` capability, show a "restart now" entry. Header field names are unified as `web.username` / `web.password` / `rtsp.*` / `onvif.*` / `gb28181.*` (device-specific extra sections are free-form).

### 5.1 Service Restart (Extension: `restart`)

| Method | Path | Description |
|------|------|------|
| POST | `/api/system/restart` | Restarts the device service process (applying all saved config with `restart` semantics). Responds `200 {"status":"restarting"}` as soon as possible, then the process exits and is re-spawned (normally within seconds). Idempotent: repeated calls during restart have no side effects |

Frontend flow: confirm dialog → POST → poll `GET /api/health` (public) until recovery → refresh the page.

## 6. Event Channel (Core: `GET /api/events`, SSE)

- `Content-Type: text/event-stream`; 15s keep-alive comment lines; auth required (the cookie is carried by EventSource automatically).
- Event format: `event: <type>\ndata: <json>\n\n`.
- Clients must not assume a fixed event set — follow `capabilities.events` and ignore unknown event types.

Vocabulary:

| type | payload | Produced by |
|------|---------|--------|
| `camera_added` | `{"camera_id","name","device_index"?}` | notebook hot-plug |
| `camera_offlined` | `{"camera_id"}` | notebook hot-plug |
| `param_changed` | `{"camera_id","name","value"}` | imaging parameter changed by any client |
| `ai_detection` | `{"camera_id","detections":[{"label","confidence","bbox"}],"frame_number"?}` | AI inference frame; bbox coordinate system as in §4.6 (video pixel space) |
| `recording` | `{"camera_id","active"}` | recording started/stopped |
| `status` | `{"uptime",...}` | periodic status summary (optional) |

## 7. Appendix A: Accepted Device Dialects (explicit difference list)

1. **Transport**: Pi uses HTTP :8088 (cookie without `Secure`); notebook defaults to TLS :8443 (cookie with `Secure`), optionally with `web.http_port` opening an extra plain-HTTP listener for certificate-free LAN access — sessions issued via that port carry cookies without `Secure` (browsers refuse to send `Secure` cookies over http://, which would make login on the HTTP port impossible). The frontend adapts via `location.protocol`.
2. **Go legacy `/snapshot`** (:8088, unauthenticated) is kept for NVR pull, coexisting with `/api/cameras/0/snapshot`.
3. **Credential storage**: Pi stores credentials in the config file (rs persists sessions to `web-sessions.json` next to the config; go does the same); notebook stores them in SQLite (bcrypt hashes, persistent sessions). No difference at the auth protocol level.
4. **Go has no MJPEG** (the H.264 pipeline has no raw frames), `capabilities.mjpeg=false`, frontend falls back to snapshot polling.
5. **metrics metric names** are device-specific (`mibee_eye_*` / `mibee_*`) and not unified.
6. **Go HLS** (`/hls/*`) and its dedicated metrics port :9100 remain Go-specific dialects.
7. **notebook protocol hot-switch**: `GET /api/protocols/runtime-status` is a notebook extension endpoint (ONVIF/GB28181/RTMP runtime state); the configuration itself lives in the `protocols` section of `/api/config`.
8. **Config file formats**: Go YAML, Pi Rust TOML, notebook SQLite — invisible to the frontend, merely the persistence layer of `PUT /api/config`.
9. **Device-level flip (hflip/vflip)**: flips are baked into the encoded stream, persistently affecting every viewer (RTSP/ONVIF/GB28181/recording/snapshot), independent of the browser-side display-only live flip button (localStorage). Config location dialects: rs uses `camera.hflip`/`camera.vflip` in `/api/config` (bool, restart to apply); Go uses the same fields (via libcamera transform, restart to apply), and Go's imaging endpoint (§4.5) additionally forwards `VFlip`/`HFlip` to `camera.vflip`/`camera.hflip` with a restart (response carries `applied:"restart"` — an explicit exception to §4.5 "immediate", since rpicam-vid has no runtime flip channel; flip requests whose value equals the current one are idempotent no-ops: nothing written, no restart, no `applied` field); notebook uses per-camera `config.hflip`/`config.vflip` on `PUT /api/cameras/{id}` (applied when the camera stream (re)starts, frontend camera card offers a flip button and automatically stops → starts).
10. **Config apply path**: Go automatically restarts the service on save (`applied:"restart"` implemented as a SIGTERM self-restart; sessions persist in `web-sessions.json` next to the config, so the browser stays logged in transparently across self-restarts from save/flip/§5.1 explicit restart; explicit logout or password reset still clears all sessions); rs only writes to disk on save, applied by an explicit `POST /api/system/restart` (§5.1) from the user (sessions likewise persist to `web-sessions.json` next to the config and survive the restart); notebook hot-applies per section and has no `restart` capability (`capabilities.restart=false`, the frontend shows no restart entry).
11. **Observability (§3.2) implementation dialects**: Go keeps the dedicated 9100 Prometheus port (legacy scraping config) while also serving `/metrics` on the web port; rs serves `/metrics` on the web port only. The log ring buffer covers the log facade (Go: full slog; rs: the protocol-library log facade); request tracing covers the Web API surface; RTSP/ONVIF/GB28181 dedicated port faces are covered by `/metrics` counters. The notebook backend does not implement §3.2 yet (`capabilities.observability` absent, the frontend hides the resource monitor automatically).

## 8. Appendix B: Legacy Endpoints Replaced by This Specification (migration table)

| Legacy endpoint (project) | New endpoint |
|----------------|--------|
| `GET /health` (go/notebook) | `GET /api/health` |
| `GET /api/version` (go) | Merged into `firmware` of `GET /api/status` |
| `POST /api/login` + token (go) | `POST /api/auth/login` + cookie |
| `X-Password` write gate (rs) | cookie session + CSRF |
| `GET /api/settings`, `/api/protocols/{x}` GET/PUT (notebook) | `GET/PUT /api/config` (`protocols` section) |
| `POST /api/config/onvif`, `/api/config/gb28181` (go) | `PUT /api/config` partial merge |
| `GET /api/camera/params` etc. (go) | `GET/POST /api/cameras/{id}/imaging/*` |
| `GET /api/stream/ws` (go), `GET /ws/video` (rs) | `GET /api/cameras/{id}/stream.mse` |
| `GET /ws` (go/rs control channel) | `GET /api/events` (SSE) |
| `GET /api/stream` (rs MJPEG), `/api/cameras/{id}/live` (notebook) | `GET /api/cameras/{id}/live` |
| `GET /api/capture`, `/snapshot.jpg` (rs) | `GET /api/cameras/{id}/snapshot` |
| `GET/POST /api/record` (rs) | `GET/POST /api/cameras/{id}/recording` |
