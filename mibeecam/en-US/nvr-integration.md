# Integrating with MiBee NVR

MiBee NVR supports multiple ingest paths: ONVIF auto-discovery, direct RTSP, RTMP/SRT push-in, GB/T 28181, and the Xiaomi camera protocol. The table below maps each MiBeeCam model to its recommended path.

## Ingest Matrix

| Camera Project | Exposed Protocols | Recommended NVR Ingest |
|----------------|-------------------|------------------------|
| Seeed XIAO ESP32-S3 Sense | MJPEG HTTP stream, NAS upload | ffmpeg relay of MJPEG to RTMP/SRT; or browse the NAS folder |
| Luatos ESP32-S3 A10 | MJPEG HTTP stream | Same — ffmpeg relay to RTMP/SRT |
| AI-Thinker ESP32-CAM | MJPEG HTTP stream, REST | Same — ffmpeg relay to RTMP/SRT |
| ESP32-S3-N16R8 | **RTSP (digest auth) + ONVIF discovery** | NVR ONVIF auto-discovery, or direct RTSP |
| rpi-cam | **Full ONVIF + RTSP + RTMP push + WS-Discovery** | NVR ONVIF auto-discovery (zero config) |

## Relaying MJPEG Devices

For ESP32 cameras that only expose an MJPEG HTTP stream, run ffmpeg on any always-on device:

```bash
ffmpeg -f mjpeg -i http://192.168.1.50:8080/stream \
  -c:v copy -f rtsp rtsp://nvr-host:8554/live/esp32-cam1
```

Push-in ingest (RTMP/SRT) and stream address configuration: NVR manual — [SRT / RTMP Streaming](https://www.mlsbs.top/docs/mibeenvr/srt-rtmp).

## Discovery and Direct Connect

- **ONVIF auto-discovery**: rpi-cam and the N16R8 support WS-Discovery / ONVIF — one-click add from the NVR web UI → [ONVIF Auto-discovery](https://www.mlsbs.top/docs/mibeenvr/onvif-discovery)
- **Direct RTSP**: the N16R8 RTSP server uses digest auth — pick RTSP when adding the camera and provide credentials
- **Raspberry Pi camera guide** → [Raspberry Pi Camera](https://www.mlsbs.top/docs/mibeenvr/raspberrypi)
- **More brand compatibility** → [Camera Brand Compatibility Guide](https://www.mlsbs.top/docs/mibeenvr/camera-guide)
