# Console tour

The console (`reach-config.exe` / `MiBeeReach.app`) has three pages —
**Connect / Workspace / Settings** — switched from the left rail. The
workspace gives the remote picture maximum vertical space; fullscreen hides
the whole rail.

## Connect page (default)

- **Quick Connect**: one box, smart routing — an `ip:port` goes over LAN
  direct; an optional **LAN access key** field carries the pairing
  credential (see [Pairing](pairing.md)).
- **Device grid**: same-segment controlled machines appear automatically
  (beacons every 3 s, 15 s freshness window) — zero accounts, zero
  configuration. Each card shows online state and address; **Connect**
  opens the desktop.
- **SSH sessions** (collapsed card): store "device + username" presets and
  open them as embedded terminal tabs.
- **Session monitor** (collapsed card): live speed / RTT / FPS / bitrate
  for every active session, 3-second refresh.

## Workspace page

All live sessions appear as tabs (remote desktop / embedded terminal / SSH).
The 36px toolbar shows only for the **active desktop tab**:

| Tool | Effect |
|---|---|
| Zoom group | Fit / stretch-fill / 100% / ±25% / fullscreen / pop out as a window |
| Monitor | Cycle the remote monitor (`Ctrl+Alt+N`) |
| Quality | Preset cycle: original → data-saver → balanced (`Ctrl+Alt+Q`) |
| Audio | Transport toggle (`Ctrl+Alt+M`; off = zero audio bandwidth) |
| Record | Recording output toggle (`Ctrl+Alt+R`; host must run with `--record`) |
| Stats | Remote stats overlay: fps / latency / bitrate (`Ctrl+Alt+I` when popped out) |

Fullscreen (`Ctrl+Alt+F` or the toolbar) collapses rail and toolbar; `Esc`
exits.

## Settings page

- **Desktop defaults** (host launch parameters): default FPS, capture
  timeout, tile rows/cols, capture scale, display scale, incoming-file
  directory, encoder backend (auto / force software / force hardware).
- **Default listen addresses**: desktop 7777 / terminal 7778 / SSH 2222 /
  tunnel 7779.
- **Tunnel & shell**: default forward target and local port, default shell.
- **UI preferences**: auto-fullscreen after connect (off by default),
  embedded terminal font size. Stored locally only.

## First-launch welcome

A one-time welcome wizard appears on the first start (pick the
**LAN direct** path to begin); it never reappears afterwards.
