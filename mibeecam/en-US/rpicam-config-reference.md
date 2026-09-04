# Configuration Reference

Default values for every section of the [configuration guide](rpicam-configuration.md), plus the environment variable override list. Ready-to-adapt complete configs are in the [configuration examples](rpicam-config-examples.md).

## Default Values Reference

| Section | Field | Default Value | Type | Description |
|---------|-------|---------------|------|-------------|
| **camera** | mode | `"mtxrpicam"` | string | Capture mode (mtxrpicam / rpicamvid / rtsp) |
| | bin_path | `"deploy/bin/mtxrpicam"` | string | Path to mtxrpicam binary |
| | device | `/dev/video0` | string | Camera device path |
| | width | `1280` | int | Capture width in pixels |
| | height | `720` | int | Capture height in pixels |
| | fps | `15` | int | Frames per second |
| | codec | `"h264"` | string | Video codec |
| | bitrate | `2000000` | int | Bitrate in bits per second |
| | brightness | `0.0` | float | Brightness control |
| | contrast | `1.0` | float | Contrast control |
| | saturation | `1.0` | float | Saturation control |
| | sharpness | `1.0` | float | Sharpness control |
| | rtsp_url | `""` | string | External RTSP URL (when mode=rtsp) |
| | idr_period | `15` | int | Keyframe interval |
| | frame_buffer_size | `30` | int | Frame channel buffer capacity |
| | max_backoff | `"30s"` | string | Maximum subprocess restart backoff |
| **rtsp** | port | `8554` | int | RTSP server port |
| | username | `""` | string | RTSP username |
| | password | `""` | string | RTSP password |
| | subscriber_buffer_size | `64` | int | AUHub subscriber channel buffer size |
| | write_queue_size | `2048` | int | gortsplib write queue size |
| | enable_udp | `true` | bool | Enable UDP transport |
| | udp_rtp_port | `8000` | int | UDP RTP port |
| | udp_rtcp_port | `8001` | int | UDP RTCP port |
| **onvif** | port | `8080` | int | ONVIF HTTP port |
| | username | `"admin"` | string | ONVIF username |
| | password | `""` | string | ONVIF password |
| **device** | name | `"Pi Camera V1"` | string | Friendly camera name |
| | manufacturer | `"Raspberry Pi"` | string | Device manufacturer |
| | model | `"OV5647"` | string | Camera sensor model |
| | firmware | `"1.0.0"` | string | Firmware version |
| | hardware_id | `"OV5647"` | string | Hardware identifier |
| | serial_number | `""` | string | Device serial number |
| **logging** | level | `"info"` | string | Log level |
| **web** | enabled | `true` | bool | Enable Web UI |
| | port | `8088` | int | Web UI HTTP port |
| | username | `""` | string | Web UI username (falls back to onvif.username) |
| | password | `""` | string | Web UI password (falls back to onvif.password) |
| | allowed_origins | `["*"]` | []string | CORS allowed origins |
| | read_header_timeout | `"5s"` | string | HTTP read header timeout |
| | read_timeout | `"10s"` | string | HTTP read timeout |
| | write_timeout | `"30s"` | string | HTTP write timeout |
| | idle_timeout | `"120s"` | string | HTTP idle timeout |
| **metrics** | enabled | `true` | bool | Enable metrics endpoint |
| | port | `9100` | int | Metrics HTTP server port |
| **snapshot** | enabled | `true` | bool | Enable snapshot endpoint |
| | quality | `85` | int | JPEG quality 1-100 |
| **rtmp** | enabled | `false` | bool | Enable RTMP push |
| | url | `""` | string | RTMP push URL |
| | max_retries | `10` | int | Maximum reconnection attempts |
| **hls** | enabled | `false` | bool | Enable HLS server |
| | segment_duration | `"2s"` | string | Target segment duration |
| **gb28181** | enabled | `false` | bool | Enable GB registration |
| | transport | `"udp"` | string | SIP transport (udp / tcp) |
| | platform_sip_address | `"192.168.1.1"` | string | Platform SIP server address |
| | platform_sip_port | `5060` | int | Platform SIP port |
| | sip_domain | `"3402000000"` | string | GB domain (10 digits) |
| | device_id | `"34020000001320000001"` | string | Device ID (20 digits) |
| | channel_id | `"34020000001320000001"` | string | Channel ID (20 digits) |
| | password | `"12345678"` | string | Access password |
| | local_sip_port | `5060` | int | Local SIP listening port |
| | register_interval_secs | `60` | int | Registration interval (seconds) |
| | heartbeat_interval_secs | `60` | int | Keep-alive interval (seconds) |
| | heartbeat_timeout_count | `3` | int | Missed heartbeats before offline |
| **recording** | enabled | `false` | bool | Enable continuous recording |
| | storage_path | `"recordings"` | string | Recording root directory |
| | segment_secs | `600` | int | Target segment duration (seconds) |
| | retention_days | `3` | int | Segment retention days (0 = keep forever) |
| | max_storage_mb | `8192` | int | Storage cap in MB (0 = unlimited) |

## Environment Variable Overrides

All configuration values can be overridden using environment variables with the `MIBEE_EYE_` prefix. This is useful for deployment, testing, and containerized environments.

### Format
Environment variables follow the pattern: `MIBEE_EYE_<SECTION>_<FIELD>`

### Examples

```bash
# Override camera resolution
MIBEE_EYE_CAMERA_WIDTH=1920 MIBEE_EYE_CAMERA_HEIGHT=1080 ./mibee-eye

# Set ONVIF password for production
MIBEE_EYE_ONVIF_PASSWORD=securepassword123 ./mibee-eye

# Change RTSP port
MIBEE_EYE_RTSP_PORT=554 ./mibee-eye

# Enable debug logging
MIBEE_EYE_LOGGING_LEVEL=debug ./mibee-eye

# Web UI access and credentials
MIBEE_EYE_WEB_ENABLED=true ./mibee-eye

# Set web UI credentials (separate from ONVIF)
MIBEE_EYE_WEB_USERNAME=admin MIBEE_EYE_WEB_PASSWORD=webpass ./mibee-eye

# Set device information
MIBEE_EYE_DEVICE_NAME="Office Camera" ./mibee-eye

# Enable GB28181 registration (example)
MIBEE_EYE_GB28181_ENABLED=true ./mibee-eye
```

### All Environment Variables

| Section | Field | Environment Variable |
|---------|-------|---------------------|
| **camera** | device | `MIBEE_EYE_CAMERA_DEVICE` |
| | width | `MIBEE_EYE_CAMERA_WIDTH` |
| | height | `MIBEE_EYE_CAMERA_HEIGHT` |
| | fps | `MIBEE_EYE_CAMERA_FPS` |
| | codec | `MIBEE_EYE_CAMERA_CODEC` |
| | bitrate | `MIBEE_EYE_CAMERA_BITRATE` |
| | brightness | `MIBEE_EYE_CAMERA_BRIGHTNESS` |
| | contrast | `MIBEE_EYE_CAMERA_CONTRAST` |
| | saturation | `MIBEE_EYE_CAMERA_SATURATION` |
| | sharpness | `MIBEE_EYE_CAMERA_SHARPNESS` |
| | idr_period | `MIBEE_EYE_CAMERA_IDR_PERIOD` |
| | bin_path | `MIBEE_EYE_CAMERA_BIN_PATH` |
| | frame_buffer_size | `MIBEE_EYE_CAMERA_FRAME_BUFFER_SIZE` |
| | max_backoff | `MIBEE_EYE_CAMERA_MAX_BACKOFF` |
| **rtsp** | port | `MIBEE_EYE_RTSP_PORT` |
| | username | `MIBEE_EYE_RTSP_USERNAME` |
| | password | `MIBEE_EYE_RTSP_PASSWORD` |
| | subscriber_buffer_size | `MIBEE_EYE_RTSP_SUBSCRIBER_BUFFER_SIZE` |
| | write_queue_size | `MIBEE_EYE_RTSP_WRITE_QUEUE_SIZE` |
| | enable_udp | `MIBEE_EYE_RTSP_ENABLE_UDP` |
| | udp_rtp_port | `MIBEE_EYE_RTSP_UDP_RTP_PORT` |
| | udp_rtcp_port | `MIBEE_EYE_RTSP_UDP_RTCP_PORT` |
| **onvif** | port | `MIBEE_EYE_ONVIF_PORT` |
| | username | `MIBEE_EYE_ONVIF_USERNAME` |
| | password | `MIBEE_EYE_ONVIF_PASSWORD` |
| **device** | name | `MIBEE_EYE_DEVICE_NAME` |
| | manufacturer | `MIBEE_EYE_DEVICE_MANUFACTURER` |
| | model | `MIBEE_EYE_DEVICE_MODEL` |
| | firmware | `MIBEE_EYE_DEVICE_FIRMWARE` |
| | hardware_id | `MIBEE_EYE_DEVICE_HARDWAREID` |
| | serial_number | `MIBEE_EYE_DEVICE_SERIALNUMBER` |
| **logging** | level | `MIBEE_EYE_LOGGING_LEVEL` |
| **web** | enabled | `MIBEE_EYE_WEB_ENABLED` |
| | port | `MIBEE_EYE_WEB_PORT` |
| | username | `MIBEE_EYE_WEB_USERNAME` |
| | password | `MIBEE_EYE_WEB_PASSWORD` |
| | allowed_origins | `MIBEE_EYE_WEB_ALLOWED_ORIGINS` |
| | read_header_timeout | `MIBEE_EYE_WEB_READ_HEADER_TIMEOUT` |
| | read_timeout | `MIBEE_EYE_WEB_READ_TIMEOUT` |
| | write_timeout | `MIBEE_EYE_WEB_WRITE_TIMEOUT` |
| | idle_timeout | `MIBEE_EYE_WEB_IDLE_TIMEOUT` |
| **metrics** | enabled | `MIBEE_EYE_METRICS_ENABLED` |
| | port | `MIBEE_EYE_METRICS_PORT` |
| **snapshot** | enabled | `MIBEE_EYE_SNAPSHOT_ENABLED` |
| | quality | `MIBEE_EYE_SNAPSHOT_QUALITY` |
| **rtmp** | enabled | `MIBEE_EYE_RTMP_ENABLED` |
| | url | `MIBEE_EYE_RTMP_URL` |
| | max_retries | `MIBEE_EYE_RTMP_MAX_RETRIES` |
| **hls** | enabled | `MIBEE_EYE_HLS_ENABLED` |
| | segment_duration | `MIBEE_EYE_HLS_SEGMENT_DURATION` |
| **gb28181** | enabled | `MIBEE_EYE_GB28181_ENABLED` |
| | transport | `MIBEE_EYE_GB28181_TRANSPORT` |
| | platform_sip_address | `MIBEE_EYE_GB28181_PLATFORM_SIP_ADDRESS` |
| | platform_sip_port | `MIBEE_EYE_GB28181_PLATFORM_SIP_PORT` |
| | sip_domain | `MIBEE_EYE_GB28181_SIP_DOMAIN` |
| | device_id | `MIBEE_EYE_GB28181_DEVICE_ID` |
| | channel_id | `MIBEE_EYE_GB28181_CHANNEL_ID` |
| | password | `MIBEE_EYE_GB28181_PASSWORD` |
| | local_sip_port | `MIBEE_EYE_GB28181_LOCAL_SIP_PORT` |
| | register_interval_secs | `MIBEE_EYE_GB28181_REGISTER_INTERVAL_SECS` |
| | heartbeat_interval_secs | `MIBEE_EYE_GB28181_HEARTBEAT_INTERVAL_SECS` |
| | heartbeat_timeout_count | `MIBEE_EYE_GB28181_HEARTBEAT_TIMEOUT_COUNT` |
| **recording** | enabled | `MIBEE_EYE_RECORDING_ENABLED` |
| | storage_path | `MIBEE_EYE_RECORDING_STORAGE_PATH` |
| | segment_secs | `MIBEE_EYE_RECORDING_SEGMENT_SECS` |
| | retention_days | `MIBEE_EYE_RECORDING_RETENTION_DAYS` |
| | max_storage_mb | `MIBEE_EYE_RECORDING_MAX_STORAGE_MB` |
