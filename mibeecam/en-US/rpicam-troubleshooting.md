# Troubleshooting

> Command examples target the **Go edition** service (systemd unit `mibee-eye`); see [Rust Edition](rpicam-rs.md) — port and protocol diagnostics apply to both.

Common issues and solutions for mibee-eye, the single-board computer ONVIF camera service. For camera / encoder / RTSP / snapshot / HLS topics see [streaming troubleshooting](rpicam-troubleshooting-media.md).

## Quick Health Check

```bash
# Check if mibee-eye is running
systemctl status mibee-eye

# Check camera device
ls -la /dev/video0

# Check network connectivity
netstat -tlnp | grep -E '8554|8080|3702|8088'

# Check memory usage
free -h

# Check CPU usage
top -bn1 | head -20

# Check web UI
curl -s -o /dev/null -w "%{http_code}" http://localhost:8088/
# Expected: 200 (frontend served)

# Check liveness probe
curl -s http://localhost:8088/api/health

# Check HLS stream
curl -s http://localhost:8088/hls/stream.m3u8 | head -5

# Check camera encoder is working (look for x264 in logs)
journalctl -u mibee-eye --since "1 minute ago" | grep -i "x264\|encoder\|h264"
```

## Web UI Login Issues

### Symptoms
- Login page shows but credentials don't work
- "Invalid credentials" error on login
- Cannot access web admin panel

### Diagnosis
```bash
# Check web UI is running
curl -s -o /dev/null -w "%{http_code}" http://localhost:8088/

# Check auth endpoint
curl -s -X POST http://localhost:8088/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"test"}'

# Check credentials in config
grep -A2 'web:' config.yaml
```

### Solutions
1. **Default credentials**: Web UI falls back to ONVIF credentials when web.username/web.password are empty. Set explicit web credentials:
   ```yaml
   web:
     username: "admin"
     password: "your-password"
   ```
2. **Session expired**: login is session-based (cookie + CSRF, see the [unified Web API spec](webui-spec.md)). Clear browser cache and sign in again.
3. **Check config**: Ensure web.enabled: true in config.yaml
4. **Restart service**: sudo systemctl restart mibee-eye

## ONVIF Discovery Issues

### Symptoms
- NVR can't discover camera
- WS-Discovery probe fails
- ONVIF device service not found

### Diagnosis
```bash
# Check ONVIF HTTP port
netstat -tlnp | grep 8080

# Test UDP multicast port
nc -ul 3702

# Check ONVIF service logs
journalctl -u mibee-eye -f

# Test ONVIF endpoint manually
curl -X POST http://localhost:8080/onvif/device_service
```

### Solutions
1. **Network issues**: Check multicast routing
   ```bash
   # Enable multicast if needed
   sudo sysctl -w net.ipv4.conf.all.mc_forwarding=1
   ```

2. **Port conflict**: Change ONVIF port
   ```yaml
   onvif:
     port: 8081  # Change if needed
   ```

3. **Firewall blocks**: Allow ONVIF ports
   ```bash
   sudo ufw allow 8080/tcp
   sudo ufw allow 3702/udp
   ```

4. **Discovery timeout**: Increase in NVR config if needed

## NVR Integration Issues

### Symptoms
- NVR shows camera but can't add
- GetStreamUri fails
- Authentication issues

### Diagnosis
```bash
# Check ONVIF credentials in config.yaml
# Test ONVIF client manually
curl -X POST -H "Content-Type: application/soap+xml" \
  -d "<soap:Envelope>...</soap:Envelope>" \
  http://localhost:8080/onvif/device_service

# Check RTSP URL format
echo "rtsp://localhost:8554/stream"

# Check device info response
curl -s http://localhost:8080/onvif/device_service | grep -i device
```

### Solutions
1. **Authentication**: Set ONVIF credentials
   ```yaml
   onvif:
     username: "admin"
     password: "your-password"
   ```

2. **Invalid RTSP URL**: Ensure URL matches config
   ```yaml
   rtsp:
     username: ""  # Leave empty if no auth
     password: ""
   ```

3. **Profile issues**: Check video encoder config
   ```yaml
   camera:
     width: 1280
     height: 720
     codec: "h264"
   ```

4. **Device info**: Update device metadata
   ```yaml
   device:
     name: "My Camera"
     manufacturer: "Raspberry Pi"
     model: "OV5647"
   ```

## GB28181 Registration Issues

### Symptoms
- Device offline on the platform
- Frequent disconnects after registration
- No media on live request

### Diagnosis
```bash
# Watch registration and keep-alive logs
journalctl -u mibee-eye -f | grep -iE "gb28181|register|keepalive"

# Check the local SIP port
netstat -ulnp | grep 5060

# Check the gb28181 field on the status view (login required)
curl -s http://localhost:8088/api/health
```

### Solutions
1. **Registration 401**: verify `sip_domain`, `device_id`, and `password` match the platform exactly
2. **Local port conflict**: change `local_sip_port` when platform and device share a host
3. **UDP playback packet loss**: switch to `gb28181.transport: tcp`
4. **Empty playback results**: local recording is disabled — see the recording integration notes in [GB28181 Integration](rpicam-gb28181.md)

## Port 9100 Conflict

### Symptoms

- Metrics endpoint fails to start
- "Address already in use" error in logs
- Service starts but metrics not accessible

### Diagnosis

```bash
# Check if port 9100 is in use
netstat -tlnp | grep 9100

# Check for prometheus-node-exporter
ps aux | grep node_exporter
```

### Root Cause

Port 9100 is used by Prometheus node_exporter by default, conflicting with mibee-eye metrics endpoint.

### Solutions

1. **Disable metrics**: Set `metrics.enabled: false` in config.yaml

2. **Change metrics port**: Use a different port:
   ```yaml
   metrics:
     enabled: true
     port: 9101  # or any available port
   ```

## WiFi Stability Issues

### Symptoms

- RTSP/HLS streams drop intermittently
- Client disconnects randomly
- Network throughput drops under load

### Root Cause

RPi 3B WiFi drops under sustained transfer load due to hardware limitations (270Mbps theoretical).

### Solutions

1. **Use Ethernet if possible**: Wired connection is more stable

2. **Lower bitrate/fps**: Reduce camera settings for WiFi:
   ```yaml
   camera:
     fps: 10
     bitrate: 1000000  # 1 Mbps
   ```

3. **Restart service to recover**: If WiFi drops:
   ```bash
   sudo systemctl restart mibee-eye
   ```

## Performance and Resource Issues

### Symptoms
- High memory usage
- Lag in video stream
- CPU overload

### Diagnosis
```bash
# Check memory usage
free -h

# Check CPU usage
top -bn1 | grep -E 'mibee-eye|mediamtx'

# Check RTSP connections
lsof -i :8554

# Check network bandwidth
ip -s link show wlan0
```

### Solutions
1. **Reduce resolution**: Lower capture settings
   ```yaml
   camera:
     width: 640
     height: 480
     fps: 10
     bitrate: 1000000  # 1 Mbps
   ```

2. **Reduce RTSP buffer sizes** (advanced): Lower subscriber_buffer_size in config.yaml
3. **Monitor resources**: Add monitoring
   ```bash
   # Monitor memory every 5 seconds
   watch -n 5 "free -h && ps aux | grep mibee-eye"
   ```

## Out of Memory (OOM) Issues

### Symptoms
- mibee-eye process killed unexpectedly
- "Killed" in journalctl logs
- dmesg shows "Out of memory" or "oom-killer"
- System becomes unresponsive

### Diagnosis
```bash
# Check for OOM kills
dmesg | grep -i "oom\|killed"

# Check memory usage history
free -h

# Check for memory-hungry processes
ps aux --sort=-%mem | head -10
```

### Root Cause
The single-board computer has limited RAM. If another process consumes excessive memory (e.g. prometheus-node-exporter-collectors' apt_info.py using 124MB), the OOM killer will terminate the largest process.

### Solutions
1. **Check for cron/periodic jobs**: Disable unnecessary timers:
   ```bash
   systemctl list-timers | grep -E "apt|collect"
   sudo systemctl disable --now prometheus-node-exporter-apt.timer
   ```
2. **Reduce camera bitrate**: Lower in config.yaml
3. **Monitor with less memory-intensive tools**: Remove prometheus-node-exporter-collectors if not needed
4. **Add swap** (last resort): 512MB swap file provides OOM cushion

## Debug Mode and Logging

### Enable Debug Logging
```bash
# Set debug level via environment
MIBEE_EYE_LOGGING_LEVEL=debug ./mibee-eye

# Or in config.yaml
logging:
  level: "debug"
```

### Log Analysis Tips
```bash
# Follow logs in real-time
journalctl -u mibee-eye -f

# Filter error messages
journalctl -u mibee-eye | grep ERROR

# Look for timeout patterns
journalctl -u mibee-eye | grep -i timeout

# Check for resource warnings
journalctl -u mibee-eye | grep -i "memory\|cpu"
```

## Common Error Messages

### Camera Access Issues
```text
ERROR: camera device not available
```
- Solution: Stop MediaMTX, check device path

### Port Already in Use
```text
ERROR: address already in use
```
- Solution: Change port in config or stop conflicting service

### Network Issues
```text
ERROR: connection refused
```
- Solution: Check firewall, network connectivity

### Memory Issues
```text
WARNING: high memory usage detected
```
- Solution: Reduce resolution, FPS, or bitrate

### Encoder Create Error
```text
camera: mtxrpicam error: encoder_create(): unable to activate output stream
```
- Cause: Bundled libcamera shared libraries missing or LD_LIBRARY_PATH not set
- Solution: See "Camera Encoder Issues" in [streaming troubleshooting](rpicam-troubleshooting-media.md)

### Shared Library Not Found
```text
error while loading shared libraries: libcamera.so.9.9: cannot open shared object file
```
- Cause: LD_LIBRARY_PATH does not include the deploy/bin/ directory
- Solution: Set `Environment=LD_LIBRARY_PATH=<deploy-path>/bin` in systemd service

### HLS Error
```text
WARNING: HLS bridge not started
```
- Solution: Ensure `hls.enabled: true` and RTSP server is running to feed AUHub subscribers

## System Status Commands

```bash
# Complete system health check
echo "=== System Status ==="
echo "Camera Device:"
ls -la /dev/video0 2>/dev/null || echo "No camera found"

echo "Services:"
systemctl is-active mibee-eye mediamtx

echo "Network:"
netstat -tlnp | grep -E '8554|8080|3702'

echo "Web UI:"
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8088/

echo "HLS Status:"
curl -s http://localhost:8088/hls/stream.m3u8 > /dev/null 2>&1 && echo "HLS active" || echo "HLS inactive"

echo "Memory:"
free -h

echo "Camera Process:"
pgrep -f mibee-eye || echo "mibee-eye not running"

echo "Conflicting Processes:"
lsof /dev/video0 2>/dev/null | grep -v mibee-eye || echo "No conflicts"

echo "Encoder Status:"
journalctl -u mibee-eye --since "5 minutes ago" | grep -i "x264\|encoder" | tail -3

echo "Library Resolution:"
LD_LIBRARY_PATH=~/mibee-eye/deploy/bin ldd ~/mibee-eye/deploy/bin/mtxrpicam 2>&1 | grep -E "found|libcamera"
```

## Contact Support

If issues persist:
1. Check logs: `journalctl -u mibee-eye`
2. Include system info: `uname -a`, `dpkg -l | grep mibee-eye`
3. Provide exact error messages and reproduction steps
4. Include configuration file (redact sensitive data)
