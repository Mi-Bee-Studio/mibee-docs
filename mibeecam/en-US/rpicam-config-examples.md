# Configuration Examples

Ready-to-adapt complete [MiBee Eye](rpicam-configuration.md) configurations. Field meanings and defaults are in the [configuration guide](rpicam-configuration.md) and [configuration reference](rpicam-config-reference.md). Adjust device paths, credentials, and addresses for your environment in every example.

> Examples use **Go edition (YAML)** syntax; the Rust edition uses TOML with the same keys and slightly different syntax.
## Basic Configuration (Default Settings)

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

## High-Resolution Configuration

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

## Cloud Streaming Configuration

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

## Low-Bandwidth Configuration

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

## GB28181 + Local Recording Configuration

Join a GB/T 28181 platform with local recording enabled (feeding platform record queries and playback). See [GB28181 Integration](rpicam-gb28181.md) for the full walkthrough.

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
