# Event Ingest (7 Entry Points)

For integrators: any monitoring system, script or IoT service. Ingest endpoints are a **stable external contract since v0.1** — fields are only ever added, never redefined. Example addresses use the documentation range `192.0.2.10`.

## Entry points at a glance

| Entry | Endpoint | Auth | Use for |
|---|---|---|---|
| **Generic webhook** | `POST /api/ingest/webhook/{token}` | per-source token in the URL | your own systems, scripts, anything that can POST JSON |
| **Alertmanager-compatible** | `POST /api/ingest/alertmanager` | LAN exposure (enable via `ingest.alertmanager: true`) | **Prometheus / Alertmanager, zero refactor** |
| **PagerDuty Events v2-compatible** | `POST /api/ingest/pagerduty` | enable via `ingest.pagerduty: true` | every tool with a built-in "Send to PagerDuty"; the Opsgenie-migration landing spot |
| **Simple JSON** (ntfy/Gotify shape) | `POST /api/ingest/simple` | enable via `ingest.simple: true` | homelab bridging ecosystems, zero refactor |
| **Grafana native** | `POST /api/ingest/grafana` | enable via `ingest.grafana: true` | Grafana unified alerting and legacy panel alerts, both shapes |
| **Bark-compatible routing** | `GET/POST /{key}`, `/{key}/{title}/{body}` | the key identifies the channel (enable via `ingest.bark_compat: true`) | existing Bark-ecosystem scripts (NAS/docker notifiers) — change one hostname |
| **MQTT** | subscribes `{prefix}/health/+`, `{prefix}/event/+` | broker account | MiBeeNVR and other ecosystem devices |

A successful ingest returns **202 `{"event_id":...}`** — it means "accepted into the pipeline", not "delivered" (check the notification history / delivery records). Ingest and dispatch are asynchronous; **the endpoint never 5xxs because a channel is down**.

## Generic webhook: the Bell envelope

```bash
curl -X POST "http://192.0.2.10:9080/api/ingest/webhook/<your-token>" \
  -H "Content-Type: application/json" \
  -d '{"type":"device_lost","severity":"error","title":"living-room camera offline",
       "body":"NVR-01 unreachable for 10 minutes","source_ref":"cam-livingroom",
       "tags":{"site":"home"},"resolved":false,
       "dedup_key":"myapp/device_lost/cam-livingroom"}'
```

| Field | Required | Notes |
|---|---|---|
| `type` / `title` | at least one | event type / title (title defaults to type) |
| `severity` | no | `info\|warning\|error\|critical`, default warning; industry synonyms tolerated (crit/down/fatal/sev1…) |
| `body` | no | body text |
| `source_ref` | no | subject id (camera id / device uuid — **a cooldown dimension**) |
| `tags` | no | arbitrary k/v; rules match on subsets |
| `resolved` | no | `true` = recovery (skips dedup/cooldown, cancels escalation) |
| `dedup_key` | no | defaults to `source/type/source_ref`; **firing and resolved share the key to pair up** |
| `snapshot_url` | no | snapshot image URL, passed through into notifications |

**Multi-project sharing & isolation**: one token per integrator, each token bound to a `source` (e.g. `mibee-steward` / `mibee-nvr`). Rules route by `source` — projects share one instance without crossing streams (events, silences, rules and escalations are all scoped by source).

## Ecosystem adaptation matrix

| System | How | Cost |
|---|---|---|
| Prometheus / Alertmanager | `webhook_configs` pointed at the alertmanager entry | **zero refactor** (firing/resolved auto-pair) |
| Grafana unified alerting | Contact point of type Alertmanager at the same URL, or the native grafana entry | **zero refactor** |
| Any script / cron / CI | curl POST to the generic webhook | one line |
| Tools with built-in PagerDuty (NetApp / OpenSearch / …) | point them at the pagerduty entry | **zero refactor** |
| Existing Bark scripts | `api.day.app/yourkey` → `http://192.0.2.10:9080/yourkey` | change a hostname |
| Zabbix 6+ | official media script in `deploy/zabbix/` | one-time setup |
| Uptime Kuma / PRTG / WeCom-bot payloads / … | thin adapter: a dozen lines mapping their JSON onto the envelope, then POST | one small script |

Thin adapter example (Uptime Kuma → Bell, ten lines of Python):

```python
import json, urllib.request
def handler(payload):  # hook into any small service / cloud function
    body = json.dumps({
        "type": "uptime_monitor", "severity": "error" if payload["heartbeat"]["status"] == 0 else "info",
        "title": f'{payload["monitor"]["name"]}: {"DOWN" if payload["heartbeat"]["status"] == 0 else "UP"}',
        "source_ref": payload["monitor"]["name"],
        "resolved": payload["heartbeat"]["status"] == 1,
    }).encode()
    urllib.request.urlopen(urllib.request.Request(
        "http://192.0.2.10:9080/api/ingest/webhook/<your-token>", body,
        {"Content-Type": "application/json"}))
```

> The adapter doesn't need to live next to Bell — any machine that can reach both sides works; Bell only cares about the envelope that arrives.

## Endpoint details in brief

- **PagerDuty v2**: payload aligned with `events.pagerduty.com/v2/enqueue`; `trigger`/`resolve` pair by `dedup_key`, `acknowledge` is recorded as info-level (visible, not noisy); responses carry PD-compatible `{"status":"success","dedup_key":...}`. Bell-side `source` is fixed to `pagerduty`.
- **Simple JSON**: body taken from `message`/`body`/`text`; severity precedence = explicit `severity` > level words in `tags` (ntfy convention) > `priority` (0-3 info / 4-6 warning / 7-10 critical, or low/default/high/urgent/max) > info; `topic` doubles as `source_ref`. `source` fixed to `simple`.
- **Grafana**: unified-alerting shape (`status: firing|resolved` + `alertUid`) and legacy panel shape (`state: alerting|ok` + `ruleId`) both recognized; `ruleTags.severity` wins. `source` fixed to `grafana`.
- **Bark-compatible**: sound/group/copy and friends pass through; unknown parameters are ignored, never an error; the key matches a channel name or the default group.
- **MQTT**: topic whitelist (`health/*`, `camera.added`, `camera.quality`, `storage.health.changed`), auto-reconnect; AI-detection events go via the NVR webhook instead.

## Verification & troubleshooting

```bash
curl http://192.0.2.10:9080/healthz                # liveness
# did the event land? (after login):
curl -H "Authorization: Bearer $TOKEN" "http://192.0.2.10:9080/api/v1/events?source=<your-source>&limit=5"
```

Wrong token → 404; malformed payload → 400; accepted but no notification → check whether a rule matches that source/type, and whether silence or L0 filtered it (a single warning-level event needs 3 consecutive points by default — test with `error`/`critical`).
