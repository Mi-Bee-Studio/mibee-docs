# FAQ

### How does it relate to the NVR's built-in browser-side AI?

They complement each other. The browser AI runs in a Web Worker and covers the live view while a page is open; MiBeeVision is a server-side process that analyzes recordings segment by segment 7×24 and retains events and snapshots, writing them back to the NVR. The two run independently with separate switches.

### Does it need a GPU?

Not for Lite — inference is CPU (ONNX Runtime); a regular x86 server or ARM device is enough. GPU acceleration (CUDA/TensorRT) and multiple cameras are Pro capabilities (planned, release date TBA).

### Are Windows / macOS supported?

No. The current platform is **Linux x86_64 / aarch64**.

### The config page opens fine, but no events ever appear?

Check the two prerequisites in order:

1. Does `ORT_DYLIB_PATH` point at the bundled `libonnxruntime-*.so`? (Without it the inference library fails to load and the pipeline cannot process segments.)
2. Are you using the `mbv_` **API key** rather than the `mbs_` web login token? (The wrong one gets a 401 and the AI status stays empty.)

If events still don't appear, confirm the transport matches your topology: remote deployments must use `http_download`; `local_file` requires sharing the NVR recording directory.

### Event write-back returns 401?

You used an `mbs_` web-session token. `mbs_` is only valid inside the browser; generate an `mbv_` key in the NVR under "Settings → AI Detection → Generate API Key" for Vision.

### How do I know the analysis chain is working?

The Vision config page status area shows "pipeline running / NVR connected", and after a few minutes recordings show `completed` in the AI status column.

### Where are results stored?

- NVR side: AI badges in the recording list + the AI events page;
- Vision local: structured events (SQLite/libSQL) + bounding-box snapshots.

### Can I analyze recordings that already exist?

Batch re-analysis of historical footage is on the Pro capability list (planned, release date TBA). Lite today continuously analyzes newly produced recordings.

### Any restrictions on commercial use?

- The software is proprietary, © MiBee Studio; the Lite edition is distributed free via the release hub.
- The bundled YOLOv8n model is licensed under Ultralytics AGPL-3.0 — using this program for detection constitutes acceptance of the AGPL-3.0 obligations. **Commercial closed-source distribution requires a separate license** (or replacing the model with a permissively licensed one).

### How do the version numbers relate?

MiBeeVision uses its own version line `mibee-vision-vX.Y.Z`, evolving independently of the MiBeeNvr `vX.Y.Z` line.
