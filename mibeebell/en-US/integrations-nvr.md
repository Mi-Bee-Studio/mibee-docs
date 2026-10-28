# MiBeeNvr Integration

MiBeeNvr (the open-source NVR) ships a built-in alert-notification sink: camera offline/freeze, storage health, motion detection, AI events, doorbell and GB28181 alarms, all POSTed as `mibee-alert/1` JSON to a webhook receiver. **MiBeeBell is its out-of-the-box receiver — direct, zero conversion.**

```
NVR ──POST /api/ingest/webhook/{token}──▶ MiBeeBell pipeline ──▶ channels (WeCom/TG/email…) + Scout
```

## 1. Register an ingest token

In `config.yaml`, give the NVR its own token (`source` attributed to `mibee-nvr`; rules match on it):

```yaml
ingest:
  webhook_tokens:
    - token: "nvr-ingest-9f3k2"     # any random string; exists only on the NVR and Bell sides
      default_source: "mibee-nvr"
```

On the NVR side (Settings → Notifications), point the sink at Bell:

```yaml
notifications:
  enabled: true
  sinks:
    - name: "bell"
      url: "http://<bell-host>:9080/api/ingest/webhook"
      token: "nvr-ingest-9f3k2"
```

Matching tokens on both sides and you're done.

**About the test button** (the NVR "send test alert" button emits a `notify.test` event that bypasses NVR-side filters — but Bell's pipeline still applies):

- test alerts default to `warning`, which **persist_filter** swallows (L0: only the 3rd point within 10 minutes passes) — for first-time connectivity checks use `error`/`critical`;
- repeated similar tests hit **simagg** and **dedup** (all `notify.test` events share one dedup key, 60s window) — vary the title and wait 60s between tests;
- to let warning-level tests through: set `persist_filter: 1` on the test rule.

## 2. What the NVR sends

```json
{
  "schema": "mibee-alert/1",
  "event_id": "<uuid>",
  "occurred_at": "2026-10-25T04:05:06Z",
  "type": "camera.connection",
  "severity": "error",
  "title": "Camera offline: Studio",
  "source_ref": "cam-fa049182",
  "dedup_key": "mibee-nvr/connection/cam-fa049182",
  "resolved": false,
  "snapshot_url": "http://<nvr-host>:9090/api/ai/events/42/snapshot?api_key=mbv_...",
  "tags": { "camera_name": "Studio" }
}
```

Bell's ingest mapping: `type/severity/title/body/tags` pass through; `source_ref` (the camera id — a cooldown/routing dimension) passes through; `dedup_key` overrides the default key; `resolved:true` pierces dedup/cooldown/silence and marks recovery.

**Event type vocabulary** (additions only, semantics frozen; unknown types fall to default rules):

| type | Meaning | Pairs |
|---|---|---|
| `camera.connection` | offline / restored | same `dedup_key`, recovery carries `resolved:true` |
| `camera.freeze` | frozen picture / restored | same |
| `camera.stream` / `camera.quality` | stream anomaly / quality switch | — |
| `camera.motion` | ONVIF motion (high-frequency; the pipeline aggregates) | — |
| `ai.detection` | AI event (carries `snapshot_url`) | — |
| `doorbell.ring` | doorbell ring → answer/decline/hang-up resolves | same `dedup_key` |
| `storage.health` | storage fault / restored | same `dedup_key` |
| `gb28181.alarm` | GB28181 device alarm | — |
| `camera.added` | camera onboarded | — |
| `notify.test` | NVR test button | — |

**Snapshots**: `snapshot_url` is a directly GET-able image URL on the NVR; AI/doorbell frames are immutable and cacheable, `camera.motion` returns the live frame. Channels and Scout can render it directly.

## 3. The recommended rule pack

Treat event classes by nature: stateful types want cooldown + escalation, high-frequency types want aggregation, recoverable types want `notify_resolved`. Replace `CHANNEL` with your channel name.

**Option A: REST import** (running instance):

```bash
BELL=http://192.0.2.10:9080
TOKEN=$(curl -s -X POST $BELL/api/v1/auth/login \
  -H 'Content-Type: application/json' -d '{"password":"<bell-admin-password>"}' | sed 's/.*"token":"\([^"]*\)".*/\1/')
CH="chat"

post_rule() { curl -s -X POST $BELL/api/v1/rules -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d "$1"; echo; }

# 1) camera offline: 5m cooldown; still down after 15m → escalate; notify on recovery
post_rule "{\"Name\":\"nvr-camera-offline\",\"Match\":{\"source\":\"mibee-nvr\",\"type\":\"camera.connection\"},\"Channels\":[\"$CH\"],\"CooldownSeconds\":300,\"EscalateAfterSeconds\":900,\"EscalateTo\":[\"critical\"],\"NotifyResolved\":true}"

# 2) frozen picture: same stateful treatment
post_rule "{\"Name\":\"nvr-camera-freeze\",\"Match\":{\"source\":\"mibee-nvr\",\"type\":\"camera.freeze\"},\"Channels\":[\"$CH\"],\"CooldownSeconds\":300,\"NotifyResolved\":true}"

# 3) motion: high-frequency — 60s dedup + 300s cooldown
post_rule "{\"Name\":\"nvr-motion\",\"Match\":{\"source\":\"mibee-nvr\",\"type\":\"camera.motion\"},\"Channels\":[\"$CH\"],\"DedupSeconds\":60,\"CooldownSeconds\":300}"

# 4) AI events: 60s dedup (snapshot rides along)
post_rule "{\"Name\":\"nvr-ai\",\"Match\":{\"source\":\"mibee-nvr\",\"type\":\"ai.detection\"},\"Channels\":[\"$CH\"],\"DedupSeconds\":60}"

# 5) doorbell: immediate (no cooldown)
post_rule "{\"Name\":\"nvr-doorbell\",\"Match\":{\"source\":\"mibee-nvr\",\"type\":\"doorbell.ring\"},\"Channels\":[\"$CH\"],\"NotifyResolved\":true}"

# 6) storage health: severe + escalation chain
post_rule "{\"Name\":\"nvr-storage\",\"Match\":{\"source\":\"mibee-nvr\",\"type\":\"storage.health\"},\"Channels\":[\"$CH\"],\"CooldownSeconds\":300,\"EscalateAfterSeconds\":900,\"EscalateTo\":[\"critical\"],\"NotifyResolved\":true}"
```

**Option B: first-boot seed** (fresh instance, empty rules table):

```yaml
rules:
  - name: nvr-camera-offline
    match: { source: mibee-nvr, type: camera.connection }
    channels: [chat]
    cooldown: 5m
    escalate_after: 15m
    escalate_to: [critical]
  - name: nvr-motion
    match: { source: mibee-nvr, type: camera.motion }
    channels: [chat]
    dedup: 60s
    cooldown: 5m
  - name: nvr-ai
    match: { source: mibee-nvr, type: ai.detection }
    channels: [chat]
    dedup: 60s
  - name: nvr-doorbell
    match: { source: mibee-nvr, type: doorbell.ring }
    channels: [chat]
  - name: nvr-storage
    match: { source: mibee-nvr, type: storage.health }
    channels: [chat]
    cooldown: 5m
    escalate_after: 15m
    escalate_to: [critical]
```

## 4. Tuning notes

- **Cooldown/dedup keys**: the cooldown key is `{rule}|{source_ref}` — offline and freeze for the same camera never interfere; that's exactly why `source_ref` is the camera id.
- **Escalation & recovery**: `resolved` events pierce every gate; if "camera recovered" pings are unwelcome, turn off `notify_resolved` on that rule (REST POST defaults to on).
- **Bursts**: the NVR already rate-limits at 30/minute; for `camera.motion` / `ai.detection` trains, the pipeline's aggregation and dedup windows are the main defense.
- **Night do-not-disturb** (example: silence motion for 8 hours):

```bash
curl -s -X POST $BELL/api/v1/silences -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"match":{"source":"mibee-nvr","type":"camera.motion"},"duration":"8h","comment":"night DND"}'
```

- **Verification**: `GET /api/v1/events?source=mibee-nvr` for the ingest side, `GET /api/v1/deliveries?notification_id=` for the channel side; the NVR's `GET /api/notifications/status` has send counters to reconcile against.
