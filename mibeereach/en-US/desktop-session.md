# Remote desktop session

## Session features

| Feature | Description |
|---|---|
| Video + input | Tiled multi-resolution encoding; weak links drop fps/bitrate smoothly and climb back on recovery |
| Clipboard | Text / image / files, both directions — just copy & paste |
| File transfer | Drag & drop or the session toolbar; the incoming directory is configured in Settings (default Downloads/MiBeeReach) |
| Audio | Controlled-side loopback capture, toolbar toggle |
| Multi-monitor | `Ctrl+Alt+N` cycles the remote monitor |
| View-only mode | Session-level permission; remote input disabled while on |
| Recording output | With the host started via `--record`, toggle remote RTSP recording (`Ctrl+Alt+R`) |
| Dark backdrop | `Ctrl+Alt+B` — blanks the remote physical display, saving bandwidth and prying eyes |

## Hotkeys

| Hotkey | Action |
|---|---|
| `Ctrl+Alt+0` | Fit picture to window |
| `Ctrl+Alt+±` | Zoom ±25% |
| `Ctrl+Alt+N` | Switch monitor |
| `Ctrl+Alt+Q` | Quality preset cycle |
| `Ctrl+Alt+M` | Audio toggle |
| `Ctrl+Alt+B` | Dark backdrop toggle |
| `Ctrl+Alt+R` | Recording output toggle |
| `Ctrl+Alt+I` | Stats overlay (fps / latency / bitrate) |
| `Ctrl+Alt+F` / `F11` | Fullscreen (`Esc` exits) |
| `Esc` | Exit fullscreen / release pointer lock |

## Stats overlay

`Ctrl+Alt+I` opens an independent overlay showing **Path** (direct/relay),
**Peer** (remote address), **Rate** (up/down), **Qual** (link quality tier
and RTT), **Frame** (fps and encode time), FPS / bitrate. On static desktops
RTT keeps refreshing at 1 Hz from the ICE layer (independent of video
frames).

## Auto-reconnect

After a session drops, the console reconnects with exponential backoff (the
tab badge shows "reconnecting"); an unreachable controlled machine surfaces
the real error first instead of waiting blindly. If the host process died it
must be restarted (the Windows installer's optional start-on-boot covers
this).

## Performance notes

- **Near-zero CPU while unwatched**: capture "parks" on static screens and
  releases the encoders;
- **Optional hardware encoding**: Settings → encoder backend
  (Windows supports AMD AMF, host needs the AMF runtime);
- **Weak-network adaptive**: bitrate deadband + periodic raise — quality
  degrades smoothly on bad links and climbs back quickly on recovery.
