# 配置文档

MiBee Eye 配置采用 YAML 格式，控制摄像头服务的所有方面，包括捕获设置、流媒体协议、GB28181 接入、本地录像和设备标识。全部键的默认值与环境变量覆盖见[配置参考](rpicam-config-reference.md)，可直接套用的完整配置见[配置示例](rpicam-config-examples.md)。

> 本页以 **Go 版（YAML）**为准。Rust 版使用 TOML 且键名一致，见 [Rust 版](rpicam-rs.md)。
## 配置文件

### 文件位置
配置默认从 `configs/config.yaml` 加载。通过复制 `configs/config.example.yaml` 并根据您的设置进行修改来创建此文件。

### 文件格式
```yaml
# 注释使用 # 符号
# 顶级部分定义功能区域
camera:        # 摄像头捕获设置
rtsp:          # RTSP 流媒体服务器
onvif:         # ONVIF 设备服务
device:        # 设备标识
logging:       # 日志配置
web:           # Web UI 配置
metrics:       # Prometheus 指标导出器
snapshot:      # JPEG 快照端点
rtmp:          # RTMP 推送流媒体
hls:           # HLS 实时流媒体
gb28181:       # GB28181 国标注册（默认关闭）
recording:     # 本地连续录像（默认关闭）
```

## 配置部分

### 摄像头配置

摄像头捕获设置控制如何从摄像头设备捕获视频帧。

```yaml
camera:
  # 捕获模式："mtxrpicam"（默认，捆绑 libcamera 的子进程）、
  # "rpicamvid"（使用系统 rpicam-vid）或 "rtsp"（消费外部 RTSP URL）
  mode: mtxrpicam

  # mtxrpicam 二进制文件路径（摄像头捕获子进程）
  # 此二进制文件及其捆绑的 libcamera 库必须存在于该路径下
  bin_path: deploy/bin/mtxrpicam

  # 摄像头设备路径（V4L2 或 libcamera）
  device: /dev/video0

  # 捕获分辨率（宽度 x 高度）
  # 支持的分辨率：640x480, 1296x972, 1920x1080, 2592x1944
  width: 1280
  height: 720

  # 每秒帧数（OV5647 传感器最大 30）
  fps: 15

  # 视频编解码器（h264 或 h265）
  codec: h264

  # 目标比特率（每秒位数）
  # 示例：2000000 = 2 Mbps
  bitrate: 2000000

  # 图像控制（硬件特定范围适用）
  # 亮度：-1.0 到 1.0（0.0 = 默认值，负值 = 更暗，正值 = 更亮）
  brightness: 0.0

  # 对比度：0.0 到 32.0（1.0 = 默认值）
  contrast: 1.0

  # 饱和度：0.0 到 32.0（1.0 = 默认值）
  saturation: 1.0

  # 锐度：0.0 到 16.0（1.0 = 默认值）
  sharpness: 1.0

  # mode=rtsp 时的外部 RTSP URL（mode=mtxrpicam 时忽略）
  rtsp_url: ""

  # 关键帧间隔（1=每帧，15=每15帧）
  idr_period: 15

  # 帧通道缓冲容量（帧数）
  frame_buffer_size: 30

  # 最大子进程重启退避持续时间
  max_backoff: 30s
```

### RTSP 配置

RTSP 服务器设置用于视频流客户端。

```yaml
rtsp:
  # RTSP 服务器端口（默认：8554）
  port: 8554

  # 可选的 RTSP 身份验证
  # 留空字符串表示无身份验证
  username: ""
  password: ""

  # AUHub 订阅者通道缓冲大小
  subscriber_buffer_size: 64

  # gortsplib 写队列大小（256默认对WiFi来说太小）
  write_queue_size: 2048

  # 启用 UDP 传输（NVR客户端需要）
  enable_udp: true

  # UDP RTP 端口（默认：8000）
  udp_rtp_port: 8000

  # UDP RTCP 端口（默认：8001）
  udp_rtcp_port: 8001
```

### ONVIF 配置

ONVIF 服务器设置，用于通过 NVR 系统进行设备发现和控制。

```yaml
onvif:
  # ONVIF HTTP/SOAP 端口（默认：8080）
  port: 8080

  # ONVIF WS-UsernameToken 身份验证
  # MiBee NVR 集成必需
  username: "admin"

  # ONVIF 密码（生产环境必须设置）
  password: ""
```

### Web UI 配置

Web UI 设置用于内置的浏览器管理面板，提供实时预览和摄像头配置功能。

```yaml
web:
  # 启用 Web 管理界面（默认：true）
  enabled: true

  # Web UI HTTP 端口（默认：8088）
  port: 8088

  # Web UI 身份验证（会话 cookie + CSRF，见统一 Web API 规范）
  # 当用户名/密码为空时回落使用 ONVIF 凭据
  username: ""
  password: ""

  # CORS 允许的来源（生产环境使用特定来源）
  allowed_origins:
    - "*"

  # HTTP 服务器超时
  read_header_timeout: 5s
  read_timeout: 10s
  write_timeout: 30s
  idle_timeout: 120s
```

### RTMP 配置

RTMP 推送设置，用于流式传输到云服务。

```yaml
rtmp:
  # 启用 RTMP 推送流媒体
  enabled: false

  # 云服务的 RTMP 推送 URL
  # 示例：
  # - rtmp://push-server/app/stream
  # - rtmp://live.twitch.tv/app/channel-key
  url: "rtmp://push-server/app/stream"

  # 最大重连尝试次数（0 = 无限制）
  max_retries: 10
```

### GB28181 配置

国标设备侧注册，用于接入 GB/T 28181 SIP 平台。完整说明见 [GB28181 接入](rpicam-gb28181.md)。

```yaml
gb28181:
  # 启用国标注册（默认：false）
  enabled: false

  # SIP 传输层：udp（默认）或 tcp
  transport: udp

  # 平台 SIP 服务器地址与端口
  platform_sip_address: "192.168.1.1"
  platform_sip_port: 5060

  # 国标域（10 位）、设备编码与通道编码（各 20 位）
  sip_domain: "3402000000"
  device_id: "34020000001320000001"
  channel_id: "34020000001320000001"

  # 接入密码（与平台一致）
  password: "12345678"

  # 本地 SIP 监听端口（与平台同机部署时必须改端口）
  local_sip_port: 5060

  # 注册与保活周期（秒）与心跳超时次数
  register_interval_secs: 60
  heartbeat_interval_secs: 60
  heartbeat_timeout_count: 3
```

### 本地录像配置

连续录像把裸 H.264 段落盘到按天/小时组织的目录，并维护 `index.jsonl` 索引，供 GB28181 录像查询与回放/下载使用。

```yaml
recording:
  # 启用连续本地录像（默认：false）
  enabled: false

  # 录像根目录
  storage_path: "recordings"

  # 目标分段时长（秒，默认：600）
  segment_secs: 600

  # 删除超过该天数的段（0 = 永不删除）
  retention_days: 3

  # 超过该容量上限（MB）时删最旧的段（0 = 无上限）
  max_storage_mb: 8192
```

### 设备配置

通过 ONVIF GetDeviceInformation 服务公开的设备信息。

```yaml
device:
  # NVR 中显示的友好摄像头名称
  name: "Pi Camera V1"

  # 设备制造商
  manufacturer: "Raspberry Pi"

  # 摄像头传感器型号
  model: "OV5647"

  # 固件版本字符串
  firmware: "1.0.0"

  # 硬件标识符
  hardware_id: "OV5647"

  # 序列号（如果不可用则为空）
  serial_number: ""
```

### 日志配置

用于调试和监控的日志设置。

```yaml
logging:
  # 日志级别（debug, info, warn, error）
  # debug：最详细，包含所有调试消息
  # info：标准操作日志
  # warn：仅警告和错误
  # error：仅错误
  level: "info"
```

### Snapshot 配置

Snapshot 端点设置，用于通过 HTTP 进行 JPEG/H.264 捕获。

```yaml
snapshot:
  # 启用 snapshot 端点（默认：true）
  enabled: true

  # JPEG 质量 1-100（仅用于 rpicam-still 子进程；H.264 IDR 回退忽略此设置）
  quality: 85
```

Snapshot 端点使用双层策略：
1. **第一层**：当摄像头空闲时，`rpicam-still` 子进程捕获真实 JPEG
2. **第二层**：当摄像头管道忙碌时，回退到存储的 H.264 IDR 帧（返回为 `video/H264`）

### Metrics 配置

Prometheus 指标导出器设置。

```yaml
metrics:
  # 启用指标 HTTP 端点（默认：true）
  enabled: true

  # 指标 HTTP 服务器端口（默认：9100）
  # 注意：9100 与 Prometheus node_exporter 冲突 — 如果两者在同一主机上运行，请更改端口或禁用
  port: 9100
```

### HLS 配置

HLS 实时流设置，用于浏览器播放。

```yaml
hls:
  # 启用 HLS 服务器（默认：false）
  enabled: false

  # 目标分段持续时间（默认：2s）
  segment_duration: 2s
```

HLS 服务器使用纯 Go MPEG-TS 分段器 — 无 ffmpeg 子进程。分段保存在内存中。

## 配置提示

1. **摄像头兼容性**：并非所有分辨率和设置都与所有摄像头模块兼容。请使用您的特定摄像头硬件测试配置。

2. **性能**：更高的分辨率和比特率需要更多的 CPU 和带宽。在树莓派 3B 上，720p @ 15fps 是推荐的平衡点。

3. **安全性**：在生产环境中，始终为 ONVIF 身份验证设置强密码。

4. **网络**：RTSP 流媒体可能消耗大量带宽。确保您的网络基础设施能够处理所选的比特率。

5. **调试**：使用 `MIBEE_EYE_LOGGING_LEVEL=debug` 来解决配置问题。

6. **环境变量**：使用环境变量存储像密码这样的敏感数据，避免将它们存储在配置文件中。

7. **验证**：服务将根据硬件约束验证配置值。无效的设置将被记录或设置为默认值。

8. **Web UI 访问**：Web 管理面板可通过 `http://<设备-ip>:8088/` 访问，使用会话登录（默认回落 ONVIF 凭据）。端点契约见[统一 Web API 规范](webui-spec.md)。

9. **摄像头二进制文件**：`bin_path` 必须指向有效的 mtxrpicam 二进制文件。该文件所在目录还必须包含捆绑的 libcamera 共享库（libcamera.so.9.9、libcamera-base.so.9.9）和 IPA 模块。详见[部署指南](rpicam-deployment.md)。
