# Deployment & Quick Start

MiBeeVision Lite ships as a single binary, downloaded from the **MiBeeNvr GitHub releases page**, free of charge.

## Download

Release page: [MiBeeNvr Releases · MiBeeVision Lite v0.1.0](https://github.com/Mi-Bee-Studio/MiBeeNvr/releases/tag/mibee-vision-v0.1.0)

| Asset | Description |
|---|---|
| `mibee-vision-lite-v0.1.0-linux-x86_64` | x86_64 binary (YOLOv8n model built in) |
| `mibee-vision-lite-v0.1.0-linux-aarch64` | ARM64 binary (YOLOv8n model built in) |
| `libonnxruntime-1.17.1-linux-x64.so` | ONNX Runtime (CPU, x86_64) |
| `libonnxruntime-1.17.1-linux-aarch64.so` | ONNX Runtime (CPU, ARM64) |
| `SHA256SUMS` | Checksums |
| `README.txt` | Bundled readme |

- Platform: **Linux x86_64 / aarch64** (no Windows / macOS builds).
- Lite limits: single camera, CPU inference; multiple cameras / GPU acceleration / batch re-analysis are Pro capabilities (planned, release date TBA).

## Two must-read prerequisites

1. **`ORT_DYLIB_PATH` must be set** to the bundled `libonnxruntime-*.so`. Without it the inference library fails to load and the pipeline cannot process segments — symptom: "the config page opens, but no events ever appear".
2. **Use the NVR's `mbv_` API key** (NVR "Settings → AI Detection → Generate API Key"), not the web login token (`mbs_` prefix) — the latter gets a 401 on event write-back.

See [MiBeeNvr Integration](nvr-integration.md).

## Quick start (ARM64 example)

```bash
# 1. Unpack the binary and the matching .so into one directory
sudo mkdir -p /opt/mibee-vision
# (place the 4 downloaded files into /opt/mibee-vision/)

# 2. Verify integrity (optional but recommended)
sha256sum -c SHA256SUMS

# 3. Run (ORT_DYLIB_PATH is not optional)
ORT_DYLIB_PATH=/opt/mibee-vision/libonnxruntime-1.17.1-linux-aarch64.so \
  /opt/mibee-vision/mibee-vision-lite-v0.1.0-linux-aarch64 serve

# 4. Open http://<host-ip>:9092 in a browser
#    enter the NVR address → paste the mbv_ key → "Test connection and fetch cameras"
#    → pick 1 camera → enter rule name and zone vertices → "Save (pipeline restarts)"
```

**Self-check**: the config page status area should show "pipeline running / NVR connected"; after a few minutes, recordings show `completed` in the AI status column — the full chain is working.

## Remote deployment

When Vision is not on the NVR host, the transport (`[transport].type`) must be **`http_download`** (set in the config page, effective on save); `local_file` requires sharing the NVR's recording storage directory, otherwise you will see "failed to open video, skipping".

## systemd service

Reference unit (adjust paths to your setup):

```ini
# /etc/systemd/system/mibee-vision.service
[Unit]
Description=MiBeeVision Lite (video AI engine)
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/opt/mibee-vision/mibee-vision-lite-v0.1.0-linux-aarch64 serve
Environment=ORT_DYLIB_PATH=/opt/mibee-vision/libonnxruntime-1.17.1-linux-aarch64.so
WorkingDirectory=/opt/mibee-vision
Restart=on-failure
RestartSec=5

# Recommended hardening: read-only everywhere except its own data dir
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/opt/mibee-vision

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mibee-vision
journalctl -u mibee-vision -f     # watch startup and segment processing logs
```

## Ports

| Port | Purpose |
|---|---|
| `:9092` | Embedded web config page (deliberately offset from MiBeeNvr `:9090` and MiBeeSteward `:8080`) |
