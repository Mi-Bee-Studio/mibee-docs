# 接入 MiBee NVR

MiBee NVR 支持多种接入方式：ONVIF 自动发现、RTSP 直连、RTMP/SRT 推流、GB/T 28181、小米摄像头协议。下表给出 MiBeeCam 家族各型号的推荐接入路径。

## 接入方式对照

| 相机项目 | 对外协议 | 推荐 NVR 接入方式 |
|----------|----------|------------------|
| Seeed XIAO ESP32-S3 Sense | MJPEG HTTP 流、NAS 上传 | MJPEG 经 ffmpeg 转推 RTMP/SRT；或 NAS 目录回看 |
| Luatos ESP32-S3 A10 | MJPEG HTTP 流 | 同上，ffmpeg 转推 RTMP/SRT |
| AI-Thinker ESP32-CAM | MJPEG HTTP 流、REST | 同上，ffmpeg 转推 RTMP/SRT |
| ESP32-S3-N16R8 | **RTSP（digest 认证）+ ONVIF 发现** | NVR ONVIF 自动发现，或 RTSP 直连 |
| rpi-cam | **ONVIF 全服务 + RTSP + RTMP 转推 + WS-Discovery** | NVR ONVIF 自动发现（零配置） |

## MJPEG 设备转推示例

仅输出 MJPEG HTTP 流的 ESP32 相机，用一台任意常驻设备跑 ffmpeg 转推即可入 NVR：

```bash
ffmpeg -f mjpeg -i http://192.168.1.50:8080/stream \
  -c:v copy -f rtsp rtsp://nvr-host:8554/live/esp32-cam1
```

推流接入（RTMP/SRT）与拉流地址配置详见 NVR 手册 [SRT/RTMP 推流接入](https://www.mlsbs.top/docs/mibeenvr/srt-rtmp)。

## 直连与自动发现

- **ONVIF 自动发现**：rpi-cam 与 N16R8 支持 WS-Discovery / ONVIF，在 NVR Web 界面一键发现添加 → [ONVIF 自动发现](https://www.mlsbs.top/docs/mibeenvr/onvif-discovery)
- **RTSP 直连**：N16R8 的 RTSP 带 digest 认证，在 NVR 添加摄像头时选择 RTSP 并填入凭据
- **树莓派相机专项指引** → [树莓派摄像头接入](https://www.mlsbs.top/docs/mibeenvr/raspberrypi)
- **更多品牌兼容性** → [摄像头品牌兼容指南](https://www.mlsbs.top/docs/mibeenvr/camera-guide)
