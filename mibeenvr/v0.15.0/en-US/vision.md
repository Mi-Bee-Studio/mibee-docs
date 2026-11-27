# MiBee Vision

MiBee Vision is the companion server-side AI analytics component for
MiBee NVR: it runs object detection on NVR recordings, generates
zone-based events and writes them back to the NVR as browsable,
searchable results.

## Editions and availability

The currently published edition is **MiBeeVision Lite (free)**,
distributed through the
[GitHub releases page](https://github.com/Mi-Bee-Studio/MiBeeNvr/releases)
of MiBee NVR — look for the release whose name starts with
`mibee-vision-`.

- **Platform**: Linux (x86_64 / ARM64); a low-power host such as a
  Raspberry Pi is enough.
- **Lite capabilities**: single camera, CPU inference; the detection
  model is built in, no extra download needed.
- **Versioning**: Vision uses its own version line (e.g.
  `mibee-vision-v0.1.0`), independent from MiBeeNVR releases; see the
  release page for licensing notes (the built-in YOLOv8n model is
  licensed under Ultralytics AGPL-3.0).

Vision can run on the same machine as the NVR or on a separate host.
Analysis happens on that separate device or server, so the NVR's own
recording performance is unaffected.

## What it does

- **Post-recording analysis** — runs object detection on finished NVR
  recordings, fully decoupled from the recording pipeline.
- **Zone events** — draw a rule zone on the picture; when a person
  enters the zone an event with a confidence score is produced
  (e.g. "zone intrusion · person") and written back to the NVR's
  **AI events page**: grouped by day, filterable by camera and event
  type, one click away from the matching recording position.
- **Web configuration page** — NVR connection, camera selection, rule
  zones, run status and recent events all live on one web page; no
  manual config editing beyond the initial setup.

![MiBeeVision Lite configuration page](images/vision-lite-config.webp)

## Setup guide

About ten minutes end to end. Requires MiBee NVR **v0.12.0 or later**.

### 1. Generate an API key on the NVR

Go to **Settings → AI Detection → MiBeeVision integration** and
generate an API key. Keys look like `mbv_…` and are **shown only
once** — store it safely. The same page lets you revoke keys at any
time.

> Note: you must use an API key generated here. The `mbs_` session
> token from the web login does not work for Vision — write-backs
> with it are rejected.

### 2. Download and unpack

From the MiBeeVision Lite release on the releases page, download the
binary and the ONNX Runtime library matching your architecture
(together with `SHA256SUMS` and `README.txt`) into one directory
(e.g. `/opt/mibee-vision/`) and make the binary executable:

```bash
cd /opt/mibee-vision
chmod +x mibee-vision-lite-v0.1.0-linux-aarch64
```

### 3. Initialize the configuration

```bash
./mibee-vision-lite-v0.1.0-linux-aarch64 config init
```

This creates `config/config.toml` in the current directory. Depending
on your deployment, apply one of the following:

- **Separate host (recommended — Vision not on the NVR machine)**:
  set `[transport]` `type` to `http_download` (Vision downloads
  recordings from the NVR over HTTP for analysis); also set
  `[source]` `camera_whitelist` to the temporary value `["pending"]`
  — Lite requires a camera to be selected before it starts, so put a
  placeholder here and pick the real camera on the web page later.
- **Same machine**: keep `local_file` and point `[source.nvr_files]`
  `dir` at the NVR's recording storage directory (set the same
  temporary `camera_whitelist` value).

### 4. Start

```bash
ORT_DYLIB_PATH=/opt/mibee-vision/libonnxruntime-1.17.1-linux-aarch64.so \
  /opt/mibee-vision/mibee-vision-lite-v0.1.0-linux-aarch64 serve
```

`ORT_DYLIB_PATH` must point at the bundled `libonnxruntime-*.so` —
without it the inference library fails to load: the configuration page
still opens, but no recording is ever processed. This is the most
common startup problem.

For unattended operation a systemd unit works well (example):

```ini
[Unit]
Description=MiBeeVision Lite
After=network-online.target

[Service]
WorkingDirectory=/opt/mibee-vision
Environment=ORT_DYLIB_PATH=/opt/mibee-vision/libonnxruntime-1.17.1-linux-aarch64.so
ExecStart=/opt/mibee-vision/mibee-vision-lite-v0.1.0-linux-aarch64 serve
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 5. Configure on the web page

Open `http://<vision-host-IP>:9092` in a browser:

1. Enter the NVR address (e.g. `http://192.0.2.10:9090`) and the API
   key from step 1, then click **Test connection & fetch cameras** —
   the camera list appears on success;
2. pick the camera to analyze under "processing camera" (Lite handles
   one camera at a time);
3. leave the confidence threshold and sampling fps at their defaults;
4. under "event zone" enter a rule name and the zone vertices
   (normalized 0–1 coordinates; the "full frame" button fills them in
   for you); the consecutive-frame count debounces detections;
5. click **Save (restarts pipeline)** — the status row showing
   "pipeline running / NVR connected" means the integration is up.

![MiBeeVision Lite configuration page](images/vision-lite-config.webp)

### 6. Verify

- On the configuration page: **pipeline running**, **NVR connected**;
- In the NVR web UI open **Dashboard → AI events**: events appear once
  someone enters the rule zone, and recordings get the "completed" AI
  badge.

![NVR AI events page](images/vision-ai-events.webp)

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Config page opens but the pipeline fails to start or never produces results | `ORT_DYLIB_PATH` is missing or does not point at the bundled `.so`; see step 4 |
| AI badges on recordings stay empty after saving | A web session token (`mbs_`) was used instead of an API key (`mbv_`), so write-backs are rejected; generate a new API key on the NVR and update it on the config page |
| Log shows "failed to open video … No such file or directory" | Vision runs on a separate host but the transport is still `local_file`; switch to `http_download` per step 3 |
| The web page reports a JSON parse error on save (missing field …) | The config file was hand-trimmed and lost required fields; delete `config/config.toml`, run `config init` again and only change the items from step 3 |

## Relation to browser-side AI detection

The NVR web UI also has a real-time detection overlay that runs in
your browser (tuning guide see [AI detection tuning](ai-detection.md)).
The two are independent: browser-side detection only affects the
overlay you see while watching, while Vision performs server-side
analysis of recordings and writes events back to the NVR — both can
be used at the same time.

## MiBee App

The companion mobile app **MiBee App** (in beta) connects to the NVR
directly over the LAN for live view and playback; it is not covered
in detail here.
