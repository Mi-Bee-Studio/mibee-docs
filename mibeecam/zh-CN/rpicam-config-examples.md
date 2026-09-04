# 配置示例

可直接套用的 [MiBee Eye](rpicam-configuration.md) 完整配置示例。各字段含义与默认值见[配置文档](rpicam-configuration.md)与[配置参考](rpicam-config-reference.md)。所有示例均需按实际环境调整设备路径、凭据与地址。

## 基本配置（默认设置）

```yaml
# configs/config.yaml
camera:
  device: /dev/video0
  width: 1280
  height: 720
  fps: 15
  codec: h264
  bitrate: 2000000
  brightness: 0.0
  contrast: 1.0
  saturation: 1.0
  sharpness: 1.0

rtsp:
  port: 8554
  username: ""
  password: ""

onvif:
  port: 8080
  username: "admin"
  password: ""

rtmp:
  enabled: false
  url: "rtmp://push-server/app/stream"

device:
  name: "Pi Camera V1"
  manufacturer: "Raspberry Pi"
  model: "OV5647"
  firmware: "1.0.0"
  hardware_id: "OV5647"
  serial_number: ""

logging:
  level: "info"

web:
  enabled: true
  port: 8088
  username: ""
  password: ""
```

## 高分辨率配置

```yaml
camera:
  device: /dev/video0
  width: 1920
  height: 1080
  fps: 25
  codec: h264
  bitrate: 4000000  # 4 Mbps
  brightness: 0.2
  contrast: 1.5
  saturation: 1.2
  sharpness: 2.0

rtsp:
  port: 8554
  username: "stream"
  password: "streampass"

onvif:
  port: 8080
  username: "admin"
  password: "onvif123"

device:
  name: "HD Security Camera"
  manufacturer: "Raspberry Pi"
  model: "OV5647"
  firmware: "2.0.0"
  hardware_id: "OV5647-HD"
  serial_number: ""

web:
  enabled: true
  port: 8088
  username: ""
  password: ""
```

## 云流媒体配置

```yaml
camera:
  width: 1280
  height: 720
  fps: 15
  codec: h264
  bitrate: 2000000

rtsp:
  port: 8554
  username: ""
  password: ""

onvif:
  port: 8080
  username: "admin"
  password: "secure123"

rtmp:
  enabled: true
  url: "rtmp://live.example.com/live/stream-key"

device:
  name: "Cloud Stream Camera"
  manufacturer: "Raspberry Pi"
  model: "OV5647"
  firmware: "1.2.0"
  hardware_id: "OV5647-CLOUD"

logging:
  level: "warn"

web:
  enabled: true
  port: 8088
  username: ""
  password: ""
```

## 低带宽配置

```yaml
camera:
  width: 640
  height: 480
  fps: 10
  codec: h264
  bitrate: 500000  # 0.5 Mbps
  brightness: 0.0
  contrast: 1.0
  saturation: 1.0
  sharpness: 1.0

rtsp:
  port: 8554
  username: ""
  password: ""

onvif:
  port: 8080
  username: "admin"
  password: "lowpass"

device:
  name: "Low Bandwidth Camera"
  manufacturer: "Raspberry Pi"
  model: "OV5647"
  firmware: "1.0.0"
  hardware_id: "OV5647-LBW"

logging:
  level: "error"

web:
  enabled: true
  port: 8088
  username: ""
  password: ""
```

## 国标接入 + 本地录像配置

接入 GB/T 28181 平台并开启本地录像（供平台录像查询与回放），详见 [GB28181 接入](rpicam-gb28181.md)。

```yaml
camera:
  width: 1280
  height: 720
  fps: 15
  codec: h264
  bitrate: 2000000

onvif:
  port: 8080
  username: "admin"
  password: "secure123"

gb28181:
  enabled: true
  transport: udp
  platform_sip_address: "192.168.1.10"
  platform_sip_port: 5060
  sip_domain: "3402000000"
  device_id: "34020000001320000001"
  channel_id: "34020000001320000001"
  password: "12345678"
  local_sip_port: 5060

recording:
  enabled: true
  storage_path: "recordings"
  segment_secs: 600
  retention_days: 3
  max_storage_mb: 8192

device:
  name: "GB Camera"
  manufacturer: "Raspberry Pi"
  model: "OV5647"

web:
  enabled: true
  port: 8088
  username: ""
  password: ""
```
