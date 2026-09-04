# MiBeeCam Camera Documentation Hub

MiBeeCam is Mi&Bee Studio's self-hosted camera product line documentation hub. It hosts **multiple merged project collections**: each collection first documents its unified architecture, API and frontend design, then branches into per-board / per-platform manuals. Pages always describe the **current version only** — no legacy pages are kept.

Collections currently hosted:

- [ESP-Cam series (ESP32 cameras)](espcam-overview.md) — one shared architecture across four ESP32 boards, plus per-board manuals
- [Raspberry Pi camera rpi-cam](rpicam-architecture.md) — ONVIF software camera in Go

> Collections use mutually exclusive slug prefixes (ESP-Cam unified pages use `espcam-` plus per-board prefixes; the Raspberry Pi collection uses `rpicam-`). New collections (e.g. mibee-rec) register a prefix per [CONVENTIONS](https://github.com/Mi-Bee-Studio/mibee-docs/blob/main/mibeecam/CONVENTIONS.md) and join in parallel.

## Family Members

| Project | Platform / Sensor | Key Capabilities | Positioning |
|---------|-------------------|------------------|-------------|
| [Seeed XIAO ESP32-S3 Sense](https://github.com/Mi-Bee-Studio/seeed-esp32s3-cam) | ESP32-S3 + OV2640/OV5640 | MJPEG, RTSP (MJPEG + G.711 audio), AVI segmented recording, NAS upload (WebDAV/HTTP), dynamic timelapse, OTA | Balanced surveillance cam |
| [Luatos ESP32-S3 A10](https://github.com/Mi-Bee-Studio/luatos-esp32s3-a10-camera) | ESP32-S3 + OV2640 | MJPEG, motion detection, WebSocket events, webhooks, ONVIF, AT commands | Event-driven compact cam (no PSRAM) |
| [AI-Thinker ESP32-CAM](https://github.com/Mi-Bee-Studio/ai-thinker-esp32-cam) | ESP32 + OV2640 | MJPEG, motion detection, SD timelapse/burst, ONVIF, dual WiFi, file manager, OTA, lightweight MPA | Low-cost entry |
| [ESP32-S3-N16R8](https://github.com/Mi-Bee-Studio/esp32s3-n16r8-cam) | ESP32-S3 + OV3660 (3MP) | MJPEG, RTSP (digest), ONVIF, face/motion/QR detection, web OTA | HD + standard protocols + AI |
| [rpi-cam](https://github.com/Mi-Bee-Studio/mibee-eye-raspi-go) | Raspberry Pi (Go) | ONVIF Device/Media/PTZ/Imaging, RTSP, RTMP/SRT push, WS-Discovery | Raspberry Pi software camera |

## Ecosystem

The four ESP-Cam boards share one **API contract (v1.2)** and one unified frontend design (the Raspberry Pi side has its own Web API SPEC v1 + zero-build reference frontend). Both connect to any NVR over ONVIF/RTSP:

```mermaid
flowchart LR
    subgraph espcam [ESP-Cam collection (contract v1.2)]
        A[Seeed XIAO]
        B[Luatos A10]
        C[AI-Thinker]
        D["N16R8 · AI detection"]
    end
    subgraph raspi [Raspberry Pi collection (SPEC v1)]
        E[rpi-cam]
        W[mibee-webui reference frontend]
    end
    espcam -->|ONVIF / RTSP / MJPEG| NVR[MiBee NVR / any ONVIF client]
    raspi -->|ONVIF / RTSP / RTMP / SRT| NVR
    W -.-> E
```

## Where to Start

- Choosing a board: [ESP-Cam board matrix and selection guide](espcam-overview.md)
- Developing: [ESP-Cam architecture](espcam-architecture.md) → [Unified API design](espcam-api.md) → [Unified frontend design](espcam-webui.md)
- Troubleshooting: start with the [ESP-Cam knowledge base](espcam-kb.md) (EMFILE reboot loops, self-heal false kills and other family-wide pitfalls), then the per-board pages
- Deep dives: [ESP32-CAM / ESP-IDF: Constraints You Can't Code Around](aicam-esp32-cam-performance.md) · [Port Separation Design](seeed-port-separation-design.md) · [rpi-cam ONVIF Compliance](rpicam-onvif-compliance.md)
- Recording platform: [Integrating with MiBee NVR](nvr-integration.md)
