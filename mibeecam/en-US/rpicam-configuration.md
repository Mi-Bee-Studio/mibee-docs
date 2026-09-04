# Configuration

The MiBee Eye configuration is written in YAML format and controls all aspects of the camera service, including capture settings, streaming protocols, GB28181 integration, local recording, and device identification. Default values and environment variable overrides for every key are in the [configuration reference](rpicam-config-reference.md); ready-to-adapt complete configs are in the [configuration examples](rpicam-config-examples.md).

## Configuration File

### File Location
Configuration is loaded from `configs/config.yaml` by default. Create this file by copying `configs/config.example.yaml` and modifying it for your setup.

### File Format
```yaml
# Comments use the # symbol
# Top-level sections define functional areas
camera:        # Camera capture settings
rtsp:          # RTSP streaming server
onvif:         # ONVIF device services
device:        # Device identification
logging:       # Logging configuration
web:           # Web UI configuration
metrics:       # Prometheus metrics exporter
snapshot:      # JPEG snapshot endpoint
rtmp:          # RTMP push streaming
hls:           # HLS live streaming
gb28181:       # GB28181 registration (disabled by default)
recording:     # Local continuous recording (disabled by default)
```

## Configuration Sections

### Camera Configuration

Camera capture settings control how video frames are captured from the camera device.

```yaml
camera:
  # Capture mode: "mtxrpicam" (default, subprocess with bundled libcamera),
  # "rpicamvid" (system rpicam-vid), or "rtsp" (consumes an external RTSP URL)
  mode: mtxrpicam

  # Path to mtxrpicam binary (camera capture subprocess)
  # This binary and its bundled libcamera libraries must be present at this path
  bin_path: deploy/bin/mtxrpicam

  # Camera device path (V4L2 or libcamera)
  device: /dev/video0

  # Capture resolution (width x height)
  # Supported resolutions: 640x480, 1296x972, 1920x1080, 2592x1944
  width: 1280
  height: 720

  # Frames per second (max 30 for OV5647 sensor)
  fps: 15

  # Video codec (h264 or h265)
  codec: h264

  # Target bitrate in bits per second
  # Example: 2000000 = 2 Mbps
  bitrate: 2000000

  # Image controls (hardware-specific ranges apply)
  # Brightness: -1.0 to 1.0 (0.0 = default, negative = darker, positive = brighter)
  brightness: 0.0

  # Contrast: 0.0 to 32.0 (1.0 = default)
  contrast: 1.0

  # Saturation: 0.0 to 32.0 (1.0 = default)
  saturation: 1.0

  # Sharpness: 0.0 to 16.0 (1.0 = default)
  sharpness: 1.0

  # External RTSP URL when mode=rtsp (ignored when mode=mtxrpicam)
  rtsp_url: ""

  # Keyframe interval (1=every frame, 15=every 15th frame)
  idr_period: 15

  # Frame channel buffer capacity (frames)
  frame_buffer_size: 30

  # Maximum subprocess restart backoff duration
  max_backoff: 30s
```

### RTSP Configuration

RTSP server settings for video streaming clients.

```yaml
rtsp:
  # RTSP server port (default: 8554)
  port: 8554

  # Optional RTSP authentication
  # Leave empty strings for no authentication
  username: ""
  password: ""

  # AUHub subscriber channel buffer size
  subscriber_buffer_size: 64

  # gortsplib write queue size (256 default too small for WiFi)
  write_queue_size: 2048

  # Enable UDP transport (needed for NVR clients)
  enable_udp: true

  # UDP RTP port (default: 8000)
  udp_rtp_port: 8000

  # UDP RTCP port (default: 8001)
  udp_rtcp_port: 8001
```

### ONVIF Configuration

ONVIF server settings for device discovery and control via NVR systems.

```yaml
onvif:
  # ONVIF HTTP/SOAP port (default: 8080)
  port: 8080

  # ONVIF WS-UsernameToken authentication
  # Required for MiBee NVR integration
  username: "admin"

  # ONVIF password (MUST be set for production)
  password: ""
```

### Web UI Configuration

Web UI settings for the built-in browser-based admin panel with live preview and camera configuration.

```yaml
web:
  # Enable Web admin UI (default: true)
  enabled: true

  # Web UI HTTP port (default: 8088)
  port: 8088

  # Web UI authentication (session cookie + CSRF, see the unified Web API spec)
  # Falls back to ONVIF credentials when username/password are empty
  username: ""
  password: ""

  # CORS allowed origins (use specific origins in production)
  allowed_origins:
    - "*"

  # HTTP server timeouts
  read_header_timeout: 5s
  read_timeout: 10s
  write_timeout: 30s
  idle_timeout: 120s
```

### RTMP Configuration

RTMP push settings for streaming to cloud services.

```yaml
rtmp:
  # Enable RTMP push streaming
  enabled: false

  # RTMP push URL for cloud services
  # Examples:
  # - rtmp://push-server/app/stream
  # - rtmp://live.twitch.tv/app/channel-key
  url: "rtmp://push-server/app/stream"

  # Maximum reconnection attempts (0 = unlimited)
  max_retries: 10
```

### GB28181 Configuration

GB device-side registration with a GB/T 28181 SIP platform. Full walkthrough in [GB28181 Integration](rpicam-gb28181.md).

```yaml
gb28181:
  # Enable GB registration (default: false)
  enabled: false

  # SIP transport: udp (default) or tcp
  transport: udp

  # Platform SIP server address and port
  platform_sip_address: "192.168.1.1"
  platform_sip_port: 5060

  # GB domain (10 digits), device ID and channel ID (20 digits each)
  sip_domain: "3402000000"
  device_id: "34020000001320000001"
  channel_id: "34020000001320000001"

  # Access password (must match the platform)
  password: "12345678"

  # Local SIP listening port (must change when co-located with the platform)
  local_sip_port: 5060

  # Registration and keep-alive intervals (seconds) and heartbeat timeout count
  register_interval_secs: 60
  heartbeat_interval_secs: 60
  heartbeat_timeout_count: 3
```

### Local Recording Configuration

Continuous recording writes bare H.264 segments into day/hour directories and maintains an `index.jsonl` index for GB28181 record queries and playback/download.

```yaml
recording:
  # Enable continuous local recording (default: false)
  enabled: false

  # Recording root directory
  storage_path: "recordings"

  # Target segment duration in seconds (default: 600)
  segment_secs: 600

  # Delete segments older than this many days (0 = keep forever)
  retention_days: 3

  # Prune oldest segments above this cap in MB (0 = unlimited)
  max_storage_mb: 8192
```

### Device Configuration

Device information exposed via ONVIF GetDeviceInformation service.

```yaml
device:
  # Friendly camera name for display in NVR
  name: "Pi Camera V1"

  # Device manufacturer
  manufacturer: "Raspberry Pi"

  # Camera sensor model
  model: "OV5647"

  # Firmware version string
  firmware: "1.0.0"

  # Hardware identifier
  hardware_id: "OV5647"

  # Serial number (empty if not available)
  serial_number: ""
```

### Logging Configuration

Logging settings for debugging and monitoring.

```yaml
logging:
  # Log level (debug, info, warn, error)
  # debug: Most verbose, includes all debug messages
  # info: Standard operational logging
  # warn: Only warnings and errors
  # error: Only errors
  level: "info"
```

### Snapshot Configuration

Snapshot endpoint settings for JPEG/H.264 capture via HTTP.

```yaml
snapshot:
  # Enable the snapshot endpoint (default: true)
  enabled: true

  # JPEG quality 1-100 (only used for rpicam-still subprocess; H.264 IDR fallback ignores this)
  quality: 85
```

The snapshot endpoint uses a dual-tier strategy:
1. **Tier 1**: `rpicam-still` subprocess captures a real JPEG when the camera is idle
2. **Tier 2**: Falls back to the stored H.264 IDR frame (returned as `video/H264`) when the camera pipeline is busy

### Metrics Configuration

Prometheus metrics exporter settings.

```yaml
metrics:
  # Enable the metrics HTTP endpoint (default: true)
  enabled: true

  # Metrics HTTP server port (default: 9100)
  # NOTE: 9100 conflicts with Prometheus node_exporter — change port or disable if both run on same host
  port: 9100
```

### HLS Configuration

HLS live streaming settings for browser playback.

```yaml
hls:
  # Enable HLS server (default: false)
  enabled: false

  # Target segment duration (default: 2s)
  segment_duration: 2s
```

The HLS server uses a pure Go MPEG-TS segmenter — no ffmpeg subprocess. Segments are kept in-memory.

## Configuration Tips

1. **Camera Compatibility**: Not all resolutions and settings work with all camera modules. Test your configuration with your specific camera hardware.

2. **Performance**: Higher resolutions and bitrates require more CPU and bandwidth. On SBCs, 720p @ 15fps is the recommended balance.

3. **Security**: Always set a strong password for ONVIF authentication in production environments.

4. **Network**: RTSP streaming can consume significant bandwidth. Ensure your network infrastructure can handle the chosen bitrate.

5. **Debugging**: Use `MIBEE_EYE_LOGGING_LEVEL=debug` to troubleshoot configuration issues.

6. **Environment Variables**: Use environment variables for sensitive data like passwords to avoid storing them in configuration files.

7. **Validation**: The service will validate configuration values against hardware constraints. Invalid settings will be logged or defaulted.

8. **Web UI Access**: The web admin panel is available at `http://<device-ip>:8088/` with session login (falling back to ONVIF credentials by default). The endpoint contract is in the [unified Web API spec](webui-spec.md).

9. **Camera Binary**: The `bin_path` must point to a valid mtxrpicam binary. The directory containing this binary must also contain the bundled libcamera shared libraries (libcamera.so.9.9, libcamera-base.so.9.9) and IPA modules. See the [deployment guide](rpicam-deployment.md) for details.
