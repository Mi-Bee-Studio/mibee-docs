# Rust Edition (Closed-Source)

Besides the open-source Go edition, MiBee Eye also ships a device edition implemented in Rust. The Rust edition is delivered as a pre-built firmware / service with **the device source code not published**; its protocol stack is built entirely on our open-source protocol libraries, and its protocol behavior maps one-to-one to those libraries.

## Positioning and Open-Source Boundary

| Layer | Availability | Notes |
|----|---------|------|
| Device service itself (Rust) | Closed-source release | Delivered as a pre-built binary; no source build provided |
| Protocol libraries (dependencies) | **Open source (MIT)** | Can be integrated into your own projects independently |

The two open-source libraries the Rust edition consumes:

| Library | Crate | Purpose | Docs |
|----|-------|------|------|
| [onvif-rs](https://github.com/mickeyzzc/onvif-rs) | `onvif-device-rs` | ONVIF Device server: WS-Discovery, Media, PTZ, Imaging, Security | [ONVIF · Rust](https://www.mlsbs.top/docs/mibeelibs/onvif-rs-discovery) |
| [gb28181-rs](https://github.com/mickeyzzc/gb28181-rs) | `gb28181-rs` | GB28181 device side: registration / keep-alive, catalog, live, playback | [GB28181 · Rust](https://www.mlsbs.top/docs/mibeelibs/gb28181-rs-server) |

In other words: the device's protocol-level interoperability behavior (wire format, state machines, response semantics) is defined and golden-test-constrained by these open-source libraries — independent of whether the device source is public. Integrators can develop against the libraries, verify compatibility, or reuse the same protocol stack for their own products. See the [library overview](https://www.mlsbs.top/docs/mibeelibs/libs-overview).

## Capability Overview

| Capability | Notes |
|------|------|
| ONVIF Profile S | Device / Media services + WS-Discovery (UDP multicast and HTTP probe paths) |
| GB28181 device side | Registration and keep-alive, catalog and device info, live streaming, record query and playback / download (UDP / TCP) |
| RTSP live | H.264 / H.265 streaming over UDP and RTP over TCP |
| Local recording | Continuous segments on disk + index, feeding GB playback / download, pruned by retention days and storage cap |
| Web admin | Unified Web API spec v1: session auth, MSE live view, config read/write, SSE events |
| Snapshot | JPEG snapshot endpoint |
| Motion detection | Frame-difference detection, runs locally |
| Native capture | Native V4L2 / libcamera bindings, no external capture subprocess |

Protocol operation guides are shared with the Go edition: ONVIF behavior in [ONVIF Compliance Reference](rpicam-onvif-compliance.md), GB28181 onboarding in [GB28181 Integration](rpicam-gb28181.md) (which includes the Rust edition's TOML example).

## Comparison With the Go Edition

| Dimension | Go edition (open source) | Rust edition (closed-source release) |
|------|--------------|-------------------|
| Source | Public at [mibee-eye-raspi-go](https://github.com/Mi-Bee-Studio/mibee-eye-raspi-go) | Not published; delivered pre-built |
| Camera capture | Bundled mtxrpicam subprocess (own libcamera) | Native V4L2 / libcamera bindings |
| ONVIF server | onvif-go/v2 | onvif-rs (`onvif-device-rs`) |
| GB28181 device side | gb28181-go | gb28181-rs |
| Config format | YAML | TOML (key names match the Go edition) |
| Web port | HTTP :8088 | HTTP :8088 |
| Config apply | Auto-restarts on save | Saved to disk, applied via an explicit `POST /api/system/restart` (login survives the restart) |

Both editions share the same [unified Web API specification](webui-spec.md) and the same embedded frontend — feature differences follow each device's `/api/capabilities` advertisement.

## Configuration and Usage

- The config file is TOML; key names, semantics, and defaults match the Go edition (`camera` / `rtsp` / `onvif` / `device` / `logging` / `web` / `metrics` / `snapshot` / `gb28181` / `recording`). For key-level reference read the [configuration guide](rpicam-configuration.md) and [configuration reference](rpicam-config-reference.md) (their examples use YAML syntax; TOML syntax differs slightly, with `[section]` headers mapping to the same keys).
- Endpoint contracts for login, config, and events are in the [unified Web API specification](webui-spec.md); on the Rust edition, saved config takes effect after an explicit restart.
- Device- and platform-side deployment is provided through the delivery channel; contact us for release packages if you want to evaluate or integrate.

## Integration and Customization

- To reuse the same protocol behavior in your own product, depend on the two open-source libraries directly — the device itself is not required.
- Report protocol compatibility / interoperability issues on the respective library's GitHub issues; contact MiBee for device feature requests.
