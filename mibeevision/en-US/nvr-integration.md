# MiBeeNvr Integration

MiBeeVision and MiBeeNvr are natural partners: the NVR produces footage, Vision produces judgment (detection / tracking / rules), and the judgment flows back into the NVR for presentation. This chapter covers the wiring and where results go.

## Three steps to wire it up

1. **Generate an API key in the NVR**: NVR "Settings → AI Detection → Generate API Key". The key starts with **`mbv_`**.
2. **Enter it in the Vision config page**: open `http://<vision-host-ip>:9092`, enter the NVR address, paste the `mbv_` key, click "Test connection and fetch cameras".
3. **Pick a camera and draw rules**: choose the camera to analyze (one in Lite), enter rule names and zone vertices, click "Save" — the pipeline restarts automatically and analysis begins.

## ⚠️ Two must-read prerequisites

Missing either one typically shows up as "the config page opens fine, but no events ever appear":

| Prerequisite | What goes wrong without it |
|---|---|
| `ORT_DYLIB_PATH` must point at the bundled `libonnxruntime-*.so` | The inference library fails to load; the pipeline cannot process segments (the config page still opens) |
| Use the **`mbv_` API key**, not the web login token (`mbs_` prefix) | Event write-back returns 401; recordings never get an AI status |

The `mbs_` token is only valid for browser sessions; the `mbv_` API key is what external components like Vision use.

## Where results flow

```
NVR recording segments ──► Vision analysis ──┬──► AI badge column in the recording list
                                             ├──► AI events (zone intrusion / crossing / loitering)
                                             └──► bounding-box snapshots
```

- **Recording list**: every recording gets an AI status badge — see at a glance what has been analyzed and what was found.
- **AI events**: rule-triggered events are visible on the NVR's AI events page.
- **Self-check**: the Vision config page status area shows "pipeline running / NVR connected", and after a few minutes recordings show `completed` in the AI status column — the full chain is working.

## Deployment topologies

| Topology | Transport (`[transport].type`) | Requirement |
|---|---|---|
| Vision on the same host as the NVR | `local_file` | Vision can read the NVR recording storage directory directly (same shared path) |
| Vision on a separate host | `http_download` | Vision can reach the NVR over the network; segments are downloaded before analysis |

Both are selected and saved in the Vision config page. If a remote deployment logs "failed to open video, skipping: No such file or directory", you are almost certainly using `local_file` without a shared storage path.

## Version pairing

MiBeeVision uses its own version line (`mibee-vision-vX.Y.Z`), unrelated to NVR version numbers. Use an MiBeeNvr version from the same era as Vision Lite or newer, so the AI status and event interfaces are available.
