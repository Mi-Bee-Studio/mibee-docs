# Introduction

**MiBeeReach** is the remote-access product of the MiBee ecosystem: **remote desktop, remote terminal, SSH and port forwarding** share one connection layer and one device identity, entirely over your own local network — **no accounts, no cloud, no third-party relay**.

In one sentence: install a **Host** (controlled side) on the machine you want to reach, open the **Console** (controller side) on the machine in your hands — same-segment devices appear automatically, one click connects. Desktop, terminal, files, clipboard, audio — all within reach.

## What it solves

| Pain point of commercial remote-desktop services | MiBeeReach's answer |
|---|---|
| Speed caps / quotas | Your own infrastructure — no caps, no quotas |
| Ads and upsells | A pure tool, no ads |
| Privacy black box (traffic through vendor servers) | Encrypted LAN-only transport — traffic never leaves your network |
| Uncontrollable quality / latency | Full in-house stack: adaptive quality, near-zero CPU while unwatched |

## Capabilities

| Capability | Description |
|---|---|
| Remote desktop | Video, keyboard & mouse, clipboard (text / image / files), audio, file transfer, multi-monitor, view-only mode, recording output |
| Remote terminal | Remote shell over the same transport |
| Remote SSH | Built-in SSH server — any standard `ssh` client connects |
| Port forwarding | Expose intranet TCP services to a controller-local port |
| Management console | Desktop app: quick connect, LAN discovery, workspace tabs, embedded terminal, SSH sessions, session monitor |

## Platforms

| Role | Platforms |
|---|---|
| Controlled side (Host) | Windows x64 · Linux x86_64 · Linux arm64 |
| Controller side (Console) | Windows x64 · macOS arm64 (Apple Silicon) |

## Design principles

1. **Privacy first**: connections are always encrypted, traffic stays on your LAN, no third party in the path.
2. **Pairing required**: credential-less connections are rejected; paired connections work without prompts.
3. **Weak-network ready**: adaptive quality, near-zero CPU while unwatched, optional hardware encoding.
4. **Session-level permissions**: control / clipboard / files authorized separately, plus view-only mode.

## Next steps

- [Quick start](quick-start.md) — first remote desktop in ten minutes
- [Console tour](console.md) — Connect / Workspace / Settings
