# Quick start

Goal: **first remote desktop within ten minutes**. Path: install the Host on
the controlled machine → open the Console on the controller → automatic
discovery → connect.

## Step 1 — controlled side: install the Host

### Windows

1. Download `MiBeeReach-Host_0.1.0_x64-setup.exe` from the
   [release page](https://github.com/mickeyzzc/releases/releases/tag/mibeereach-v0.1.0).
2. Run the installer (per-user install to `%LOCALAPPDATA%\MiBeeReach-Host`,
   optional start-on-boot).
3. The host lives in the system tray and listens on `0.0.0.0:7777`.

### Linux (x86_64 / arm64)

```sh
tar xzf mibeereach_linux_<arch>.tar.gz
./reach desktop host --bind 0.0.0.0:7777 --fps 20
```

To keep it running across sessions, wrap the command in your service manager
of choice (systemd user unit, tmux, rc.local, …).

## Step 2 — controller side: install the Console

### Windows

Download `mibeereach_windows_amd64.zip`, extract anywhere, run
`reach-config.exe`. (`reach.exe` next to it is the engine the console
launches for sessions — **keep both files in the same folder**.)

### macOS (Apple Silicon)

Download `mibeereach_macos_aarch64.tar.gz`, extract, and copy
`MiBeeReach.app` (console) and `MiBeeReach Viewer.app` to **Applications**.
If Gatekeeper blocks the first launch: right-click → **Open** (regular
notice for ad-hoc signed bundles).

## Step 3 — connect

1. Open the console — it lands on the **Connect** page.
2. Controlled machines on the same network segment **appear automatically**
   in the device grid within seconds. Click **Connect**.
3. Not listed? Type the address into **Quick Connect** as `ip:port`
   (e.g. `192.168.1.10:7777`) and click **Connect**.
4. The remote desktop opens as a tab on the **Workspace** page — keyboard,
   mouse, clipboard, files and audio work right away.

## Default ports

| Service | Port | Protocol |
|---|---|---|
| Desktop host | 7777 | UDP (QUIC) |
| Terminal host | 7778 | UDP (QUIC) |
| Tunnel host | 7779 | UDP (QUIC) |

Allow the corresponding **UDP** port through the controlled machine's
firewall; check here first when connections fail.

## Next steps

- [Console tour](console.md) — what each page does
- [Pairing & LAN access key](pairing.md) — control who may connect
- [Remote desktop session](desktop-session.md) — hotkeys and feature matrix
