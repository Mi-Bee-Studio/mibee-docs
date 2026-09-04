# ESP-Cam Development Knowledge Base

The pitfalls the four repositories have hit, distilled into symptom → root cause → fix. These entries come from real debugging sessions (maintained internally in per-repo `AGENTS.md` and a cross-repo pitfall log; this page is the public, redacted edition). Before debugging any "WiFi unstable / device reboots / web broken" report, match the signature here first — then suspect the hardware.

## Network & Stability

### The EMFILE reboot loop — the #1 cause of "unstable WiFi"

**Symptom**: pages work intermittently, HTTP resets, jittery ping — looks like an RF problem.

**Root-cause chain**: the browser MJPEG watchdog reconnects every ~7s → every kicked connection lingers in TIME_WAIT on the device (lwIP default 2×MSL = 120s) → the default `LWIP_MAX_SOCKETS=10` overflows → httpd `accept` fails with EMFILE → the health probe fails repeatedly → self-heal reboots → the loop re-arms ~12s after boot.

**Log signature** (all three together = confirmed):

```
E httpd: httpd_accept_conn: error in accept (23)
W health_monitor: httpd :80 probe failed (n/6)
rst:0xc (SW_CPU_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
```

**Fix**: `CONFIG_LWIP_MAX_SOCKETS=16` (24 on seeed) + `CONFIG_LWIP_TCP_MSL=15000` in `sdkconfig.defaults` (TIME_WAIT shrinks to 30s). After changing defaults you must delete the generated sdkconfig and reconfigure, or the new values silently never apply. Landed on all four repos.

**Lesson**: on `accept (23)`, check the socket table before blaming the antenna; ping jitter measured during a reboot loop carries no diagnostic value.

### Self-heal false kills: probe failure ≠ dead device

Any "probe failed → reboot" logic must exclude two false-kill classes before rebooting: **network unreachable** (local probes always fail when WiFi is down) and **EMFILE** (the probe itself needs a socket; a full table makes it fail). Rule: don't count while WiFi is disconnected; fix the EMFILE root cause instead of loosening the reboot threshold.

### "The board is slow" — check which AP it joined first

**Symptom**: one board takes seconds to load pages with ~hundreds-of-ms ping while another board on the same LAN streams at 6ms ping.

**Root cause (three layers stacked)**: ① joined a 40MHz (HT40) 2.4GHz AP — occupies two channels, ~6dB worse sensitivity, degraded PER at weak signal; ② AMPDU aggregation disabled (an old workaround) collapses throughput to 1-3Mbps so MJPEG monopolizes airtime and ping/HTTP queue behind it; ③ association-time RSSI is untrustworthy (-56 at association, -70 steady-state).

**Fix**: force STA to 20MHz + re-enable AMPDU (current config on all S3 boards). The ESP32 log line `connected with <ssid>, channel N, 40D|BW20` reveals every link parameter at once. Cross-board latency comparison is the sharpest localization tool: when one board is fast and others slow, compare what they share (AP, channel/bandwidth, aggregation) before touching firmware.

### S3 heat is an independent failure source

When the chip temperature stays above spec (logs reporting 90°C+ every 60s), every latency problem is amplified. Check `/api/status` `chip_temp` before concluding anything; sustained streaming under high heat significantly degrades network behavior even though firmware can't fix the heat itself.

## Memory & Concurrency

### Single-framebuffer board constraints (no-PSRAM boards)

- Two MJPEG clients crush the heap to byte-levels → the stream task dies. Fix: hard single-viewer limit (LRU-kick the old viewer) + heap-watermark defense-in-depth.
- **A high early-boot heap reading bypasses the watermark gate** — the gate is defense-in-depth, never the only gate.
- Hot camera reconfig (deinit+init) racing concurrent frame grabs is fatal: no-PSRAM boards use "save → ack → reboot app after 1s". PSRAM boards (fb_count=2) can reconfigure live. **Not interchangeable between boards — don't copy.**

### Hidden compile-time caps for events/timers

Stacked features silently exceed compile-time tables and partially fail: the event-bus subscription table, httpd `max_uri_handlers`, the lwIP socket table are the same class of trap. Count table capacity before adding subscribers. **A wildcard route registered first shadows exact endpoints** (`GET /*` makes `/api/camera` 404 at runtime) — registration order: exact → `/ws` → static fallback.

### Use esp_timer for uptime, not time(NULL)

After SNTP sync, `time(NULL)` jumps to epoch (logs show astronomically large "Uptime"). All durations use `esp_timer_get_time()`.

## Build & Firmware Engineering

### sdkconfig.defaults changes don't apply

`sdkconfig` is generated; changing defaults without deleting it means new values **silently never apply**. Iron rule: change defaults → `rm sdkconfig` → set-target → build → grep the generated file to confirm.

### Frontend is build-coupled to firmware

`main/web_ui/` assets are packed into SPIFFS at build time — HTML/JS/CSS changes need a rebuild + reflash; the CMake spiffs image needs explicit per-file `DEPENDS` (directory-level deps don't trigger on edits).

### Never hand-edit managed_components

Changes there die on `fullclean`/fresh clone and silently fork CI from local builds. Legitimate paths: `patches/` + root-CMake copy step, or vendoring into `components/`.

### Shared SPA four-file discipline

`index.html / app.js / i18n.js / style.css` must be md5-identical across the three S3 repos: change one, sync the other two, verify. Feature differences degrade via capability probing — never per-board forks (see [frontend design](espcam-webui.md)).

## Hardware & Platform

### Trust the device over the docs

Production batches exist where the docs said OV2640 but the board carried an OV5640 (auto-detected by the driver). The sensor/config truth is always the boot log or the `camera` field of `/api/status`; report contradictions instead of "correcting" the device to match docs.

### ai-thinker (original ESP32) specifics

- Camera must initialize after the STA associates (earlier init triggers a DMA freeze)
- GPIO14 is a shared camera/SD bus: runtime SD formatting deadlocks → format at boot after a reboot
- GPIO0 is XCLK and cannot be a button
- Full PHY calibration on every boot; erase the phy_init partition when it goes wrong
- RTS stuck in reset after flashing: `esptool --no-stub run` releases it

### n16r8 (Octal PSRAM) specifics

- 8MB **Octal** PSRAM (not Quad): `CONFIG_SPIRAM_MODE_OCT=y`
- The 64B cache line is mandatory for Octal DDR mode — 32B causes silent data corruption
- esp32-camera changes go through the `patches/` mechanism

### luatos (no PSRAM by design) specifics

- PSRAM disabled **by design** — enabling it boot-loops
- Pinned to ESP-IDF v5.5.4 (the other three repos are on v6.0.1)
- Single app partition: no OTA capability; upgrades go over serial
- WPA3 off, STA forced to HT20, hard single-viewer limit (see single-framebuffer constraints)

## Integration & Operations

### Web OTA takes a raw binary stream

The OTA endpoints are not multipart forms — they take `Content-Type: application/octet-stream` raw streams (curl examples in [Unified API](espcam-api.md)). Images must fit the OTA slot; a mid-upload SPIFFS failure requires serial rescue; verify by the `running_partition` flip + the device's `/app.js` md5 matching the repo.

**Day-to-day upgrades go over Web OTA first** (on OTA-capable boards): delivering every change through OTA keeps both the firmware and the UI upload paths continuously working; USB flashing is reserved for blank chips, rescue, or unreachable devices. luatos has no OTA (single partition — serial is the only path); n16r8's OTA endpoints are in development, USB until they land.

### Stable device identity comes from eFuse

Serial/UUID derive from the factory eFuse MAC, read once and cached. The historical bug read the WiFi MAC at request time — all zeros before WiFi was ready, polluting NVR-side stable_id bookkeeping.

## Further Reading

Per-repo deep dives: [ESP32-CAM / ESP-IDF: Constraints You Can't Code Around](aicam-esp32-cam-performance.md), [AI-Thinker performance notes](aicam-performance.md). Wiring and per-board fixes: each board manual's troubleshooting page.
