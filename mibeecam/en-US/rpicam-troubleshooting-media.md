# Streaming Troubleshooting

> This page targets the **Go edition** capture chain (mtxrpicam); the Rust edition captures via native V4L2/libcamera with no mtxrpicam subprocess, so encoder-specific items do not apply. Other diagnostics are shared — see [Rust Edition](rpicam-rs.md).

Symptoms, diagnosis, and solutions for MiBee Eye camera detection, encoder, RTSP, snapshot, and HLS issues. For web UI / ONVIF / NVR / system-resource topics see [troubleshooting](rpicam-troubleshooting.md); for GB28181 see [GB28181 Integration](rpicam-gb28181.md).

## Camera Detection Issues

### Symptoms
- Camera not found in NVR discovery
- mibee-eye logs show "camera not detected"
- Stream shows black screen

### Diagnosis
```bash
# Check if camera device exists
ls -la /dev/video0
# Should show /dev/video0 character device

# Test camera with libcamera directly
rpicam-hello

# Check device tree overlay
cat /boot/firmware/config.txt | grep dtoverlay
# Should show: dtoverlay=ov5647

# Check kernel camera support (ignore if device works via libcamera)
vcgencmd get_camera
```

### Solutions
1. **MediaMTX conflicts**: Stop MediaMTX first
   ```bash
   sudo systemctl stop mediamtx
   sudo systemctl disable mediamtx
   ```

2. **Missing DT overlay**: Add to config.txt
   ```bash
   sudo nano /boot/firmware/config.txt
   # Add: dtoverlay=ov5647
   sudo reboot
   ```

3. **Camera module not connected**: Check CSI cable

4. **Wrong device path**: Update config.yaml
   ```yaml
   camera:
     device: "/dev/video0"  # or your camera device
   ```

## Camera Encoder Issues

### Symptoms
- Service starts but logs show "encoder_create(): unable to activate output stream"
- RTSP stream connects but no video data
- Snapshot endpoint returns 503 Service Unavailable

### Diagnosis
```bash
# Check if mtxrpicam can find libcamera
LD_LIBRARY_PATH=/home/pi/mibee-eye/deploy/bin ldd ~/mibee-eye/deploy/bin/mtxrpicam
# If "libcamera.so.9.9 => not found", bundled libs are missing

# Check if bundled libcamera files exist
ls -la ~/mibee-eye/deploy/bin/libcamera*.so*

# Check LD_LIBRARY_PATH in systemd service
grep LD_LIBRARY_PATH /etc/systemd/system/mibee-eye.service

# Test camera directly with rpicam-vid
rpicam-vid -t 1000 --width 1280 --height 720 -o /dev/null

# Check system libcamera version (may differ from bundled version)
dpkg -l | grep libcamera
```

### Root Cause
The `mtxrpicam` binary is dynamically linked against `libcamera.so.9.9`, which is NOT the same as the system-installed libcamera (Debian 13 provides `libcamera.so.0.7`). The bundled version must be present in `deploy/bin/` and `LD_LIBRARY_PATH` must point there.

### Solutions
1. **Missing bundled libs**: Re-deploy from mediamtx-rpicamera release
   ```bash
   # Download and extract on workstation
   gh release download v2.6.0 --repo bluenviron/mediamtx-rpicamera \
     --pattern "mtxrpicam_64.tar.gz"
   tar xzf mtxrpicam_64.tar.gz

   # Copy bundled libs to device
   scp mtxrpicam_64/libcamera*.so* <your-rpi-user>@<your-rpi-ip>:~/mibee-eye/deploy/bin/
   scp mtxrpicam_64/mtxrpicam <your-rpi-user>@<your-rpi-ip>:~/mibee-eye/deploy/bin/

   # Restart service
   sudo systemctl restart mibee-eye
   ```

2. **LD_LIBRARY_PATH not set**: Verify systemd service configuration
   ```bash
   # Should contain: Environment=LD_LIBRARY_PATH=/path/to/deploy/bin
   systemctl cat mibee-eye

   # If missing, edit the service file
   sudo systemctl edit mibee-eye --force
   # Add: Environment=LD_LIBRARY_PATH=/home/pi/mibee-eye/deploy/bin
   sudo systemctl daemon-reload
   sudo systemctl restart mibee-eye
   ```

3. **Camera held by another process**: Stop MediaMTX
   ```bash
   sudo systemctl stop mediamtx
   sudo systemctl disable mediamtx
   # Verify camera is free
   lsof /dev/video0
   ```

## RTSP Streaming Issues

### Symptoms
- RTSP stream not accessible
- Connection timeouts
- NVR can't connect to stream

### Diagnosis
```bash
# Check RTSP port is listening
netstat -tlnp | grep 8554

# Test RTSP connection locally
ffplay rtsp://localhost:8554/stream

# Check firewall rules
sudo ufw status

# Check camera exclusivity
lsof /dev/video0
```

### Solutions
1. **Port conflict**: Change RTSP port in config.yaml
   ```yaml
   rtsp:
     port: 8555  # Change if needed
   ```

2. **Firewall block**: Allow RTSP port
   ```bash
   sudo ufw allow 8554/tcp
   ```

3. **Camera access conflict**: Ensure only one process uses /dev/video0
   ```bash
   sudo systemctl stop mediamtx  # if running
   ```

4. **Network issues**: Check from client system
   ```bash
   telnet <camera-ip> 8554
   ```

## Snapshot Issues

### Symptoms
- Snapshot endpoint returns error
- NVR can't capture images

The snapshot endpoint lives on the web port :8088 (the legacy unauthenticated `/snapshot` plus the authenticated `/api/cameras/0/snapshot`, see the [unified Web API spec](webui-spec.md)).

### Diagnosis
```bash
# Check snapshot endpoint
curl -I http://localhost:8088/snapshot

# Test snapshot endpoint and check response
curl -s -w "\nHTTP Status: %{http_code}\n" http://localhost:8088/snapshot -o /dev/null
# HTTP 200 + "image/jpeg" = working
# HTTP 503 = camera not providing frames (check encoder)
```

### Solutions
1. **Camera not running**: Ensure mibee-eye is active
   ```bash
   sudo systemctl restart mibee-eye
   ```

2. **Resolution issues**: Adjust camera resolution in config.yaml
   ```yaml
   camera:
     width: 1280
     height: 720
   ```

3. **Encoder not producing frames**: Check logs and verify mtxrpicam is working
   ```bash
   journalctl -u mibee-eye --since "5 minutes ago" | grep -i "encoder\|h264"
   ```

## HLS Live Preview Issues

### Symptoms
- Web UI shows black video player
- "HLS not available" message in web UI
- Browser console shows hls.js errors

### Diagnosis
```bash
# Check HLS HTTP endpoint
curl -s http://localhost:8088/hls/stream.m3u8

# Verify HLS is enabled in config
grep -A2 'hls:' config.yaml

# Check mibee-eye logs for HLS errors
journalctl -u mibee-eye --grep "HLS"

# Check RTSP stream is feeding HLS (must be running)
journalctl -u mibee-eye --grep "AUHub"
```

### Solutions
1. **HLS not enabled**: Enable HLS in config.yaml
   ```yaml
   hls:
     enabled: true
     segment_duration: 2s
   ```

2. **RTSP source unavailable**: Ensure RTSP stream is working first:
   ```bash
   ffprobe rtsp://localhost:8554/stream
   ```

3. **Browser compatibility**: HLS uses pure Go MPEG-TS segmenter served via HTTP. Ensure browser supports MSE or use hls.js library.

4. **Restart mibee-eye**: sudo systemctl restart mibee-eye

5. **Debug logging**: Set `logging.level: debug` to see segmenter errors
