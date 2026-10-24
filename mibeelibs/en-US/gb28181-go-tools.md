# Release tools: device-register / platform-uas / psmux-rtp

The three runnable examples in `examples/` double as **interop tools** —
every [release](https://github.com/mickeyzzc/gb28181-go/releases) ships
them as prebuilt bundles for seven platforms
(`gb28181-go-tools_<os>_<arch>.tar.gz`, `.zip` on Windows, plus
`SHA256SUMS`): linux amd64/arm64/arm, darwin amd64/arm64, windows
amd64/arm64. Built with `CGO_ENABLED=0` — static, zero-dependency;
unpack and run on a Pi, a NAS, or a test laptop, no Go toolchain
needed.

## Which tool for which job?

| You want to… | Run |
|---|---|
| Exercise a GB/T 28181 platform's device-facing behavior without a real camera | `device-register` |
| Give a device / cascade client a minimal platform to register and pull from | `platform-uas` |
| Sanity-check the PS-mux + RTP media path (the same code path the device role uses for live push) | `psmux-rtp` |
| See how the pieces integrate (each tool is a runnable minimal integration) | all three sources |

Division of labor with the [onvif-go CLIs](onvif-go-cli.md): those are
**operations/diagnostics** tools (discover, quick-check, deep
diagnostics, virtual camera); these are **protocol-role emulators** —
each library ships tools for its own stack.

## Install

Prebuilt bundles from a release:

```bash
gh release download v0.13.0 -R mickeyzzc/gb28181-go \
  -p 'gb28181-go-tools_linux_amd64.tar.gz'
tar xzf gb28181-go-tools_linux_arm64.tar.gz   # all three binaries inside
sha256sum -c SHA256SUMS                        # verify
```

Or install from source, version-anchored (the examples are packages in
the module):

```bash
go install github.com/mickeyzzc/gb28181-go/examples/device-register@v0.13.0
```

Or from a checkout: `go run ./examples/<tool>`.

## device-register — the device (UAC) emulator

A "virtual camera": REGISTERs with the platform (digest auth) →
keepalives on a cycle → answers Catalog / DeviceInfo queries → once the
platform INVITEs, pushes RTP/PS at a **1 IDR + 1 P frame per second**
cadence (synthetic frames; no capture hardware).

Configuration lives in `main.go`'s `cfg` (no CLI flags yet): platform
address/port, device & channel 20-digit IDs, `Password` (the digest
secret agreed with the platform — **the placeholder must be changed**),
and `Transport` (`udp` | `tcp` | `tls` — SIPS, GB/T 28181-2022).

Typical uses: platform-side joint testing and regression (catalog
pagination, INVITE, TCP/SIPS transports). It is also the minimal
integration reference for the `device` package — swap the synthetic
frame loop for your encoder output feeding `device.FrameHub` and you
have production wiring.

## platform-uas — the minimal platform (UAS) server

The half devices register into: a SIP listener on `:5060`
(`ServerID 34020000002000000001`, realm `3402000000`, digest password —
change the placeholder), keepalive liveness window, catalog handling,
INVITE-driven streaming into a FrameHub fan-out, media ports
`30000-30050` (UDP), and alarms surfacing through an `EventBus`
subscription. Device/channel persistence is the `DeviceStore` seam —
the example ships an in-memory implementation; a real host writes
through to its database there.

Typical uses: give `device-register`, the cascade client, or your own
device implementation a live counterpart — no real platform needed
when exercising device-side behavior.

## psmux-rtp — the media-path self-check

A fully self-contained loopback: muxes synthetic H.264 access units
into MPEG-PS, packetizes as RTP (PT 96) toward an in-process UDP
receiver, and prints packet/byte counters every 50 packets. It listens
on nothing and needs no counterpart — one command proves the `psmux`
muxing and RTP packetization path works on the current platform (the
device role's live push and the cascade role's stream forwarding both
ride it).

Typical uses: binary smoke-test on a new arch/OS; the minimal
reproducer when chasing media-path issues.

## Common notes

- Read the sources first: these are examples (integration samples)
  before they are tools; edit-and-run (`go run`) is the intended
  configuration flow.
- The password placeholder (`your-platform-password`) must be changed
  before any interop succeeds.
- The tools evolve with releases; per-version deltas live in the repo
  [CHANGELOG](https://github.com/mickeyzzc/gb28181-go/blob/main/CHANGELOG.md).
