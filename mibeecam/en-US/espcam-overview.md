# ESP-Cam Series: Unified ESP32 Camera Architecture

ESP-Cam is the **ESP32 super-collection** of the MiBeeCam family: four boards, four independent repositories, but one shared firmware architecture, one API contract and one frontend design. This page gives the top-level view — the unified layers first, then pointers into each board's specifics.

If you just want to pick a board, read the selection table below. To understand why the four repos are "loosely coupled, tightly aligned", start with [Unified Architecture](espcam-architecture.md).

## Board Matrix

| Board | Chip / Sensor | Flash / PSRAM | Unique capabilities | Repository |
|--------|---------------|---------------|---------------------|------------|
| AI-Thinker ESP32-CAM | ESP32 (original) + OV2640 | 4 MB / none | Dual WiFi config, SD file management, lightweight MPA frontend | [ai-thinker-esp32-cam](https://github.com/Mi-Bee-Studio/ai-thinker-esp32-cam) |
| ESP32-S3 N16R8 | ESP32-S3 + OV3660 (3MP) | 16 MB / 8 MB Octal | AI pipeline (face/motion/QR), RTSP digest auth, AT commands | [esp32s3-n16r8-cam](https://github.com/Mi-Bee-Studio/esp32s3-n16r8-cam) |
| Luatos ESP32-S3 A10 | ESP32-S3 + OV2640 | 16 MB / none (by design) | WebSocket event push, webhooks, serial AT config | [luatos-esp32s3-a10-camera](https://github.com/Mi-Bee-Studio/luatos-esp32s3-a10-camera) |
| Seeed XIAO ESP32-S3 Sense | ESP32-S3 + OV2640/OV5640* | 8 MB / 8 MB Octal | AVI segmented recording, NAS upload, G.711 audio, OTA | [seeed-esp32s3-cam](https://github.com/Mi-Bee-Studio/seeed-esp32s3-cam) |

*Trust the device over the docs: some production batches shipped with an OV5640; the driver auto-detects it and `GET /api/status` reports the actual `camera` value.

## Choosing a Board

- **Lowest-cost entry**: AI-Thinker ESP32-CAM — the most mature ecosystem and notes; no PSRAM means a single stream client, and the lightest first paint on weak WiFi.
- **Need AI detection**: ESP32-S3 N16R8 — esp-dl face detection + motion + QR decoding on a dedicated core, with the best sensor (3MP).
- **Daily surveillance**: Seeed XIAO ESP32-S3 Sense — plenty of PSRAM, SD recording + NAS upload + audio monitoring out of the box, up to 3 stream clients.
- **Event-driven automation**: Luatos A10 — real-time WebSocket events (motion/recording/network) for home-automation hooks.

## Shared Design Baselines

1. **Unified API contract** ([contract v1.2](espcam-api.md)) — core endpoints 100% identical across boards; differences only ever surface through capability gating + dynamic metadata.
2. **Unified frontend** ([frontend design](espcam-webui.md)) — the three S3 boards share one SPA (four files, md5-identical); ai-thinker keeps a lightweight MPA on the same contract.
3. **Unified streaming stack** — MJPEG (`:81/stream`) + RTSP (`:554`, digest) + ONVIF (WS-Discovery + SOAP), all built on one publisher/subscriber frame broadcaster.
4. **Unified identity & config** — serial/UUID derived from the factory eFuse MAC (WiFi-state independent); NVS-backed versioned config blobs with auto-reset on magic/version mismatch.

```mermaid
flowchart TB
    subgraph repos [Four independent repositories]
        AI[ai-thinker<br/>ESP32 + OV2640]
        N1[n16r8<br/>S3 + OV3660 + AI]
        LU[luatos A10<br/>S3 + event push]
        SE[seeed XIAO<br/>S3 + recording/audio]
    end
    subgraph shared [Shared design (not a shared repo)]
        CT[API contract v1.2]
        UI[Unified SPA / MPA]
        ST[Streaming stack<br/>MJPEG + RTSP + ONVIF]
        ID[device_id / NVS config]
    end
    repos --> shared
    shared --> NVR[MiBee NVR / any ONVIF client]
```

## Going Deeper per Board

- [AI-Thinker ESP32-CAM capabilities](aicam-capabilities.md)
- [N16R8 architecture](n16r8-architecture.md)
- [Luatos A10 overview](luatos-overview.md)
- [Seeed XIAO architecture](seeed-architecture.md)

Field-tested debugging experience (EMFILE reboot loops, self-heal false kills, PSRAM constraints…) lives in the [development knowledge base](espcam-kb.md).
