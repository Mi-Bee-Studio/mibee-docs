# Architecture

MiBeeVision uses a **layered pipeline architecture**: `Source → Pipeline → Sink`, each layer behind an interface and composed through configuration.

```
┌─────────────────────────────────────────────────────────────┐
│ MiBeeVision (serve daemon + embedded web config page :9092) │
│                                                             │
│   Source layer (obtaining recording segments)               │
│   • local_file     shared recording directory with the NVR  │
│   • http_download  download segments from the NVR (remote)  │
│                 │                                           │
│                 ▼                                           │
│   Pipeline layer (analysis)                                 │
│   decode → YOLOv8n detect → ByteTrack track → zone rules    │
│                 │                                           │
│                 ▼                                           │
│   Sink layer (results)                                      │
│   • event store (SQLite/libSQL)                             │
│   • bounding-box snapshots on disk                          │
│   • write-back to MiBeeNvr (AI status + events)             │
└─────────────────────────────────────────────────────────────┘
```

## Layer responsibilities

| Layer | Responsibility | Notes |
|---|---|---|
| **Source** | Produce recording segments | Two acquisition modes by deployment shape: `local_file` when co-located with the NVR (shared recording directory); `http_download` when on a separate host (segments downloaded before analysis). Changed from the config page, effective on save |
| **Pipeline** | Turn frames into judgment | FFmpeg decode + frame extraction → YOLOv8n detection (ONNX Runtime) → ByteTrack tracking → zone intrusion / line crossing / loitering evaluation |
| **Sink** | Turn judgment into records | Events into SQLite/libSQL; a bounding-box snapshot per event; status and events written back to the NVR |

## Runtime shape

- **serve daemon**: the recommended shape today — the process runs persistently and consumes newly produced NVR recording segments; the config page (`:9092`) handles wiring (NVR address, API key, camera, rules) and a run-state self-check.
- **Configuration-driven**: camera selection, rule zones and transport mode are all edited in the config page; saving restarts the pipeline automatically — no hand-editing of config files.

## Design principles

1. **Decoupled from the NVR**: Vision is an independent process — same machine (shared storage) or a separate one (HTTP download). Either side can go down without taking the other with it.
2. **Replaceable at every layer**: segment sources, detection models and event outputs all sit behind interfaces — swapping a model or adding an output channel means "add an implementation", not "modify the pipeline".
3. **Zero cloud dependency**: inference, rules and storage all happen locally; no external service calls.

## Relationship with MiBeeNvr

The NVR is the data and semantic hub: it decides which recording segments Vision analyzes, and consumes what Vision concludes (AI status, events, snapshots). The full integration is described in [MiBeeNvr Integration](nvr-integration.md).
