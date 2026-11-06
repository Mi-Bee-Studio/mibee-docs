# Feature Overview

The table below lists what the current release (Lite v0.1.0) delivers, plus the planned Pro deltas. Everything is operated from the embedded web config page (`:9092`); changes take effect on save.

| Capability | Description | Lite v0.1.0 |
|---|---|---|
| Object detection | Built-in YOLOv8n model (ONNX Runtime, CPU), recognizes common objects (people, vehicles, animals, …) | ✅ |
| Multi-object tracking | ByteTrack cross-frame association with stable IDs and trajectories — no double-counting the same target | ✅ |
| Zone intrusion | Draw a polygon per camera in the config page; an event fires when a target enters | ✅ |
| Line crossing | Draw a boundary line; crossing direction is evaluated | ✅ |
| Loitering | Fires when a target lingers in a zone too long — perimeters, entrances | ✅ |
| Events & snapshots | Structured events stored (SQLite/libSQL), each with a bounding-box snapshot | ✅ |
| NVR write-back | Analysis status and events written back to MiBeeNvr: AI badge column in the recording list, events visible in the NVR | ✅ |
| Web config page | Enter NVR address + API key → fetch cameras → pick one → draw rules → save (pipeline restarts automatically) | ✅ |
| Multiple cameras | Analyze several streams at once | Pro |
| GPU acceleration | CUDA / TensorRT execution providers | Pro |
| Batch re-analysis | Re-run analysis over existing historical footage | Pro |

## Object detection

- Model: **YOLOv8n**, built into the binary — no separate download or configuration.
- Inference: ONNX Runtime (CPU); the matching shared library (`libonnxruntime-*.so`) ships with the release and is loaded via `ORT_DYLIB_PATH`.
- Input: frames decoded from NVR recording segments.

## Multi-object tracking

Detection boxes are handed to a **ByteTrack** tracker for cross-frame association: the same target keeps the same ID across consecutive frames, forming a trajectory. Tracking is what turns "a box on every frame" into "where this target came from, where it went, how long it stayed in the zone" — loitering and line-crossing direction are built on top of it.

## Event rules

All three rule types are configured per camera in the config page (rule name + zone vertices):

| Rule | Trigger | Typical scenario |
|---|---|---|
| Zone intrusion | A target enters the drawn area | Yard, storefront, warehouse |
| Line crossing | A target crosses the drawn boundary | Fence, gate, corridor |
| Loitering | A target stays in the area too long | Perimeter, doorway, ATM |

## Events & snapshots

- Events carry camera, time, rule type and target information, written to a local SQLite/libSQL store.
- Every event includes a **snapshot with bounding boxes drawn**, so you can see at a glance what triggered it.

## Writing back to MiBeeNvr

Results don't just stay inside Vision:

- **Recording list AI badges**: each recording shows its analysis status (in progress / completed) right in the NVR recording list.
- **AI events**: rule-triggered events are written back and visible on the NVR's AI events page.
- See [MiBeeNvr Integration](nvr-integration.md).
