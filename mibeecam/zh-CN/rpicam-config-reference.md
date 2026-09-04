# 配置参考

[配置文档](rpicam-configuration.md)中各配置节的默认值速查表与环境变量覆盖清单。可直接套用的完整配置见[配置示例](rpicam-config-examples.md)。

## 默认值参考

| 部分 | 字段 | 默认值 | 类型 | 描述 |
|------|------|--------|------|------|
| **camera** | mode | `"mtxrpicam"` | string | 捕获模式（mtxrpicam / rpicamvid / rtsp） |
| | bin_path | `"deploy/bin/mtxrpicam"` | string | mtxrpicam 二进制文件路径 |
| | device | `/dev/video0` | string | 摄像头设备路径 |
| | width | `1280` | int | 捕获宽度（像素） |
| | height | `720` | int | 捕获高度（像素） |
| | fps | `15` | int | 每秒帧数 |
| | codec | `"h264"` | string | 视频编解码器 |
| | bitrate | `2000000` | int | 比特率（每秒位数） |
| | brightness | `0.0` | float | 亮度控制 |
| | contrast | `1.0` | float | 对比度控制 |
| | saturation | `1.0` | float | 饱和度控制 |
| | sharpness | `1.0` | float | 锐度控制 |
| | rtsp_url | `""` | string | 外部 RTSP URL（mode=rtsp 时使用） |
| | idr_period | `15` | int | 关键帧间隔 |
| | frame_buffer_size | `30` | int | 帧通道缓冲容量 |
| | max_backoff | `"30s"` | string | 最大子进程重启退避 |
| **rtsp** | port | `8554` | int | RTSP 服务器端口 |
| | username | `""` | string | RTSP 用户名 |
| | password | `""` | string | RTSP 密码 |
| | subscriber_buffer_size | `64` | int | AUHub 订阅者通道缓冲大小 |
| | write_queue_size | `2048` | int | gortsplib 写队列大小 |
| | enable_udp | `true` | bool | 启用 UDP 传输 |
| | udp_rtp_port | `8000` | int | UDP RTP 端口 |
| | udp_rtcp_port | `8001` | int | UDP RTCP 端口 |
| **onvif** | port | `8080` | int | ONVIF HTTP 端口 |
| | username | `"admin"` | string | ONVIF 用户名 |
| | password | `""` | string | ONVIF 密码 |
| **device** | name | `"Pi Camera V1"` | string | 友好摄像头名称 |
| | manufacturer | `"Raspberry Pi"` | string | 设备制造商 |
| | model | `"OV5647"` | string | 摄像头传感器型号 |
| | firmware | `"1.0.0"` | string | 固件版本 |
| | hardware_id | `"OV5647"` | string | 硬件标识符 |
| | serial_number | `""` | string | 设备序列号 |
| **logging** | level | `"info"` | string | 日志级别 |
| **web** | enabled | `true` | bool | 启用 Web UI |
| | port | `8088` | int | Web UI HTTP 端口 |
| | username | `""` | string | Web UI 用户名（默认回落 onvif.username） |
| | password | `""` | string | Web UI 密码（默认回落 onvif.password） |
| | allowed_origins | `["*"]` | []string | CORS 允许的来源 |
| | read_header_timeout | `"5s"` | string | HTTP 读取头超时 |
| | read_timeout | `"10s"` | string | HTTP 读取超时 |
| | write_timeout | `"30s"` | string | HTTP 写入超时 |
| | idle_timeout | `"120s"` | string | HTTP 空闲超时 |
| **metrics** | enabled | `true` | bool | 启用指标端点 |
| | port | `9100` | int | 指标 HTTP 服务器端口 |
| **snapshot** | enabled | `true` | bool | 启用 snapshot 端点 |
| | quality | `85` | int | JPEG 质量 1-100 |
| **rtmp** | enabled | `false` | bool | 启用 RTMP 推送 |
| | url | `""` | string | RTMP 推送 URL |
| | max_retries | `10` | int | 最大重连尝试次数 |
| **hls** | enabled | `false` | bool | 启用 HLS 服务器 |
| | segment_duration | `"2s"` | string | 目标分段持续时间 |
| **gb28181** | enabled | `false` | bool | 启用国标注册 |
| | transport | `"udp"` | string | SIP 传输层（udp / tcp） |
| | platform_sip_address | `"192.168.1.1"` | string | 平台 SIP 服务器地址 |
| | platform_sip_port | `5060` | int | 平台 SIP 端口 |
| | sip_domain | `"3402000000"` | string | 国标域（10 位） |
| | device_id | `"34020000001320000001"` | string | 设备编码（20 位） |
| | channel_id | `"34020000001320000001"` | string | 通道编码（20 位） |
| | password | `"12345678"` | string | 接入密码 |
| | local_sip_port | `5060` | int | 本地 SIP 监听端口 |
| | register_interval_secs | `60` | int | 注册周期（秒） |
| | heartbeat_interval_secs | `60` | int | 保活周期（秒） |
| | heartbeat_timeout_count | `3` | int | 判定离线的心跳丢失次数 |
| **recording** | enabled | `false` | bool | 启用连续录像 |
| | storage_path | `"recordings"` | string | 录像根目录 |
| | segment_secs | `600` | int | 目标分段时长（秒） |
| | retention_days | `3` | int | 段保留天数（0 = 永不删除） |
| | max_storage_mb | `8192` | int | 存储上限 MB（0 = 无上限） |

## 环境变量覆盖

所有配置值都可以使用 `MIBEE_EYE_` 前缀的环境变量覆盖。这对于部署、测试和容器化环境很有用。

### 格式
环境变量遵循模式：`MIBEE_EYE_<部分>_<字段>`

### 示例
```bash
# 覆盖摄像头分辨率
MIBEE_EYE_CAMERA_WIDTH=1920 MIBEE_EYE_CAMERA_HEIGHT=1080 ./mibee-eye

# 为生产环境设置 ONVIF 密码
MIBEE_EYE_ONVIF_PASSWORD=securepassword123 ./mibee-eye

# 更改 RTSP 端口
MIBEE_EYE_RTSP_PORT=554 ./mibee-eye

# 启用调试日志
MIBEE_EYE_LOGGING_LEVEL=debug ./mibee-eye

# Web UI 访问和凭据
MIBEE_EYE_WEB_ENABLED=true ./mibee-eye

# 设置 Web UI 凭据（独立于 ONVIF）
MIBEE_EYE_WEB_USERNAME=admin MIBEE_EYE_WEB_PASSWORD=webpass ./mibee-eye

# 设置设备信息
MIBEE_EYE_DEVICE_NAME="Office Camera" ./mibee-eye

# 启用国标注册（示例）
MIBEE_EYE_GB28181_ENABLED=true ./mibee-eye
```

### 所有环境变量

| 部分 | 字段 | 环境变量 |
|------|------|----------|
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
