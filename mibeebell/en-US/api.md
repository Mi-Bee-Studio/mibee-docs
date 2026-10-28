# API Reference

Conventions: except ingest endpoints and admin login, all `/api/v1/*` require `Authorization: Bearer <JWT>`. Errors are uniformly `{"error":{"code":"...","message":"..."}}`; timestamps are unix milliseconds; lists are newest-first with `?limit=` returning the most recent N.

## Ingest endpoints (no JWT)

| Method | Path | Description |
|---|---|---|
| POST | `/api/ingest/webhook/{token}` | generic event envelope (fields in [Event Ingest](ingest.md)); `source` comes from the token binding |
| POST | `/api/ingest/alertmanager` | native Alertmanager webhook JSON |
| POST | `/api/ingest/pagerduty` | native PagerDuty Events v2 |
| POST | `/api/ingest/simple` | simple JSON (ntfy/Gotify shape, lenient fields) |
| POST | `/api/ingest/grafana` | Grafana unified / legacy panel alert shapes |
| GET/POST | `/{key}`, `/{key}/{title}/{body}` | Bark-compatible routing (parameters pass through) |
| GET | `/healthz` | liveness + version |
| GET | `/metrics` | Prometheus (no JWT; LAN surface) |

Response semantics: a successful ingest = **202 `{"event_id":"..."}`**, meaning accepted into the pipeline only; ingest and dispatch are asynchronous — the endpoint never 5xxs because a channel failed.

## Admin endpoints

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/auth/login` | `{"password":"..."}` → `{"token":JWT,"stream_token":"..."}` |
| POST | `/api/v1/auth/stream-token` | refresh the SSE short ticket |
| GET | `/api/v1/events` | `?since=&until=&source=&type=&severity=&dedup_key=&limit=` |
| GET | `/api/v1/notifications` | `?since=&severity=&limit=`; includes aggregated delivery status |
| GET | `/api/v1/deliveries?notification_id=` | per-attempt delivery audit |
| GET/POST/PUT/DELETE | `/api/v1/rules[/{id}]` | rule CRUD (match / channels / dedup / cooldown / escalate / persist_filter / notify_resolved / priority) |
| GET/POST/PUT/DELETE | `/api/v1/channels[/{id}]` | channel CRUD; credentials masked `***` in responses |
| POST | `/api/v1/channels/{name}/test` | send a test notification ("MiBeeBell test" + current time) |
| GET/POST/DELETE | `/api/v1/silences[/{id}]` | silence windows. POST accepts: absolute windows, `{"dedup_key":"...","duration":"1h"}` one-tap silences, and `{"cron":"23:00-06:00"}` **daily windows** (crossing midnight supported) |
| POST | `/api/v1/events/{id}/feedback` | `{"action":"confirm\|false_positive\|ack"}` |
| GET | `/api/v1/mesh` | cluster observability (members/status/claims/silence sync) |
| GET | `/api/v1/extensions` | enumerate registered channels/processors/analyzers (auditable extensibility) |

## SSE stream

```
GET /api/stream?token=<stream_token>&since=<unix-ms>
```

- `stream_token`: returned by login, valid 5 minutes, refreshable (EventSource can't carry headers, hence the separate short ticket).
- Event types (the SSE `event:` field):
  - `notification` — new notification (title/body/severity/snapshot_url; the Scout app consumes the same stream)
  - `delivery` — delivery state changes (channel ok/degraded)
  - `silence` / `escalation` — windows engaging / escalations firing (live UI badges)
- Reconnects: clients send `Last-Event-ID` as a **request header**; the server keeps an in-memory ring buffer of the last 512 notifications.

## Scout notification-center API

The stable contract for the MiBeeScout app (shipped in M8):

- `GET /api/v1/notifications?since=&unread=1` — paged fetch; `unread=1` returns only unread; rows carry `SnapshotURL` / `Source` / `SourceRef` (**stable, promised fields**) and `ReadAt` (null = unread).
- `POST /api/v1/notifications/{id}/read` — mark read (idempotent; unknown id 404).
- The SSE `notification` event serializes the same row shape.
- The Scout deep-link protocol (tap → NVR playback) is defined on the Scout side; Bell guarantees `snapshot_url` and the source-reference fields.

## Compatibility promises

- **Ingest endpoints are a stable external contract since v0.1**: fields are only added, semantics never change.
- `/api/v1/*` may break during v0.x; frozen from v1.0.
