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
| GET | `/api/v1/auth/setup-status` | first-run probe (no auth): `{"needs_setup":true}` = no password yet; the Web shows the setup form accordingly |
| POST | `/api/v1/auth/setup` | first-run password set (no auth, accepted only while no password exists): `{"password":"... ≥8 chars"}` → `{"token","stream_token"}`, same contract as login; configured instances get 403 `already_configured` |
| POST | `/api/v1/auth/login` | `{"password":"..."}` → `{"token":JWT,"stream_token":"..."}`; 5 consecutive failures from one client lock it out for 15 minutes |
| POST | `/api/v1/auth/password` | change password (hot): `{"current_password","new_password"}` → JWT signing key rotates, **all sessions invalidated immediately** |
| POST | `/api/v1/auth/stream-token` | refresh the SSE short ticket |
| GET | `/api/v1/events` | `?since=&until=&source=&type=&severity=&dedup_key=&limit=` |
| GET | `/api/v1/notifications` | `?since=&severity=&unread=&limit=`; includes aggregated delivery status |
| POST | `/api/v1/notifications/{id}/read` | mark as read (idempotent) |
| GET | `/api/v1/deliveries?notification_id=` | per-attempt delivery audit |
| GET | `/api/v1/deliveries/stats?hours=N` | per-channel delivery counts (default 24h, max 30d; feeds the health towers) |
| GET/POST/PUT/DELETE | `/api/v1/rules[/{id}]` | rule CRUD (match / channels / dedup / cooldown / escalate / persist_filter / notify_resolved / priority) |
| GET/POST/PUT/DELETE | `/api/v1/channels[/{id}]` | channel CRUD; credentials masked `***` in responses |
| POST | `/api/v1/channels/{name}/test` | send a test notification ("MiBeeBell test" + current time) |
| GET/POST/PUT/DELETE | `/api/v1/ingest-tokens[/{id}]` | ingest token management (ADR-034): add/remove takes effect **immediately, no restart**; POST returns the full token exactly once |
| GET/POST/DELETE | `/api/v1/silences[/{id}]` | silence windows. POST accepts: absolute windows, `{"dedup_key":"...","duration":"1h"}` one-tap silences, and `{"cron":"23:00-06:00"}` **daily windows** (crossing midnight supported) |
| POST | `/api/v1/events/{id}/feedback` | `{"action":"confirm\|false_positive\|ack"}` |
| GET | `/api/v1/ai/status` | AI health: analyzer state/breakers/budget + live counters of the AI pipeline-intervention gates |
| GET/PUT | `/api/v1/settings/ai` · `/telemetry` · `/ingest` · `/render` · `/retention` | settings sections (AI / OTel / ingest toggles / render / retention); masked fields sent back as `***` keep the stored value; most changes apply after restart |
| GET | `/api/v1/mesh` | cluster observability (members/status/claims/silence sync) |
| GET | `/api/v1/extensions` | enumerate registered channels/processors/analyzers (auditable extensibility) |

### Notification object (full example)

Entries of `GET /api/v1/notifications` share the same shape as SSE `notification` events:

```json
{
  "ID": "d7e00e14-4e09-430e-a928-fd04c51a2788",
  "EventID": "7fa7f6ed-bee6-4a18-912e-d07b615ef6ae",
  "RuleID": 1,
  "Severity": "warning",
  "Title": "Server CPU temperature too high",
  "Body": "CPU 82°C exceeds the 75°C threshold for 5 minutes",
  "Lang": "zh-CN",
  "Escalated": false,
  "CreatedAt": "2026-11-04T19:45:48.602+08:00",
  "SnapshotURL": "",
  "Source": "server-monitor",
  "SourceRef": "nvr-host",
  "ReadAt": null,
  "AIStatus": "",
  "TraceID": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

`ReadAt` null = unread; `AIStatus` empty = healthy (non-empty means AI was degraded when this notification went out); `TraceID` maps to the OTel pipeline trace (empty when telemetry is off).

## SSE stream

```
GET /api/stream?token=<stream_token>&since=<unix-ms>
```

- `stream_token`: returned by login, valid 5 minutes, refreshable (EventSource can't carry headers, hence the separate short ticket).
- Event types (the SSE `event:` field):
  - `notification` — new notification (title/body/severity/snapshot_url; the Scout app consumes the same stream)
  - `delivery` — delivery state changes (channel ok/degraded)
  - `silence` / `escalation` — windows engaging / escalations firing (live UI badges)
  - `ai_gate` — AI pipeline-intervention audit (ADR-036: every decision, shadow or executed; consumed live by the AI node in the flow graph)
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
