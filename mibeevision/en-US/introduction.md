# Product Introduction

**MiBeeVision** is the video-AI post-processing engine of the MiBee ecosystem — the **"AI brain" of MiBeeNvr**: it runs **object detection, multi-object tracking and zone rules** over recordings produced by the NVR, producing structured events and bounding-box snapshots that are written straight back to the NVR.

In one sentence: the NVR makes sure footage is captured; MiBeeVision makes sure it is *understood* — every recording is analyzed segment by segment as it appears, gets an AI badge in the recording list, and things like a person entering a zone, crossing a line or loitering become searchable events.

## How it differs from the NVR's built-in browser AI

MiBeeNvr ships a browser-side AI detector (inference runs in a Web Worker). The two are **complementary, not replacements**:

| | Browser-side AI (built into NVR) | MiBeeVision (server-side) |
|---|---|---|
| Where it runs | Web Worker inside your browser | Standalone process (same machine as NVR, or a separate host) |
| When it analyzes | The live view, while the page is open | Every recording segment, 7×24, page open or not |
| Event retention | Primarily live overlays | Structured events in a database + bounding-box snapshots, fully searchable |
| Rules | ROI polygon (detect inside an area) | Event rules: zone intrusion / line crossing / loitering |
| Where results go | The live view | Back to the NVR: AI badges in the recording list + AI events |

## Lite / Pro editions

MiBeeVision uses its own version line (`mibee-vision-vX.Y.Z`), independent of the NVR's `vX.Y.Z`.

| | **Lite (released · free)** | **Pro (planned; release date TBA)** |
|---|---|---|
| Current version | v0.1.0 (published 2026-10) | — |
| Cameras | Single camera | Multiple cameras |
| Inference | CPU (ONNX Runtime) | GPU acceleration (CUDA / TensorRT) |
| Analysis | Segment-by-segment as recordings appear | Plus batch re-analysis of historical footage |
| Distribution | Free, distributed via the release hub ([mickeyzzc/releases](https://github.com/mickeyzzc/releases)) | TBA |

See the [feature overview](features.md) for what the current version delivers, and [Deployment](deployment.md) to get started.

## License

- **The software** is proprietary, © MiBee Studio. The Lite edition is distributed free of charge via the release hub.
- **The bundled detection model (YOLOv8n)** is licensed under Ultralytics **AGPL-3.0** — using this program for detection constitutes acceptance of the AGPL-3.0 obligations. **Commercial closed-source distribution requires a separate license** (or replacing the model with a permissively licensed one).

## Technical profile

- **Single Rust binary**: Linux x86_64 / aarch64, no external dependencies (the ONNX Runtime shared library ships with the release).
- **ONNX Runtime inference**: CPU to start, with the YOLOv8n detection model built in.
- **FFmpeg decoding**: consumes NVR recording segments directly, no transcoding needed.
- **ByteTrack multi-object tracking**: stable IDs and trajectories across frames.
- **Embedded web config page** (default port `:9092`): enter the NVR address and API key, pick a camera, draw rule zones — save and it takes effect, no hand-editing of config files.

## Position in the ecosystem

```
Cameras → MiBeeNvr (ingest / recording / storage hub)
              │ recording segments (shared dir, or HTTP download)
              ▼
        MiBeeVision (detect → track → rule evaluation)
              │ events + snapshots + AI status written back
              ▼
        MiBeeNvr (AI badges in recording list + AI events)
```
