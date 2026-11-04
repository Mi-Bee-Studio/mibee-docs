# CLI reference

`reach` is a single multi-call binary: controlled-side and controller-side
capabilities live in one executable. Console operations end up running these
commands under the hood.

## Command map

```
reach desktop  host | connect | monitors | lan-key     Remote desktop
reach terminal host | connect                          Remote terminal
reach tunnel   host | connect                          Port forwarding
reach ssh                                              SSH server (Windows/Linux)
reach vdd       ...                                    Virtual display management (Windows only)
reach login / devices / enroll                         Platform account system (cross-network edition)
reach --version                                        Version
```

> `reach login / devices / enroll` belong to the cross-network (P2P)
> edition; the LAN edition uses LAN discovery and pairing.

## desktop

```sh
# Controlled side (most flags have sensible defaults)
reach desktop host --bind 0.0.0.0:7777 --fps 20
    --lan-key <hex64>        LAN access key (default: persisted key)
    --lan-open               Open mode (any LAN machine may connect)
    --view-only              View-only host (remote input disabled)
    --tile-cols / --tile-rows / --capture-scale   Encoding tiles & capture scale
    --audio                  Enable audio capture
    --record ...             Recording output (RTSP/ONVIF, see --help)
    --virtual-display ...    Virtual display policy (off/fallback/on, Windows only)
    --dark-backdrop          Dark backdrop policy
    --file-dir <path>        Incoming-file directory

# Controller side
reach desktop connect --host <ip>:7777
    --lan-key <hex64>        The same key as the controlled side
    --zoom fit|100|50%       Initial display scale
    --control-pipe <name>    Control pipe (console embedding)
    --embedded               Embedded mode (window geometry managed by the console)

reach desktop monitors --host <ip>:7777     List remote monitors
reach desktop lan-key                       Show / generate the persisted key
```

## terminal

```sh
reach terminal host    --bind 0.0.0.0:7778 [--lan-key <hex64>]
reach terminal connect --host <ip>:7778 [--lan-key <hex64>]
```

## tunnel

```sh
reach tunnel host    --bind 0.0.0.0:7779 [--lan-key <hex64>]
reach tunnel connect --host <ip>:7779 --local 127.0.0.1:2223 --target 127.0.0.1:22
```

`--target` is interpreted from the controlled machine's perspective; when
connecting to a desktop host (7777) each stream carries a target header and
is forwarded per-stream by the host.

## Environment variables (common)

| Variable | Purpose |
|---|---|
| `RUST_LOG` | Log level (`RUST_LOG=info` / `off`) |
| `MIBEE_ADVERTISE_ADDR` | Public endpoint advertisement (cross-network scenarios; not needed in the LAN edition) |

The full registry lives in `docs/env-vars.md` (source repository).
