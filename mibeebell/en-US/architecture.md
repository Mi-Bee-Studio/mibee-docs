# Architecture

## One-sentence architecture

A single-process Go binary: **ingest layer (seven entry kinds) → event bus (bounded, drop-oldest) → alert pipeline (silence/dedup/cooldown/escalation + L0 algorithms) → dispatcher (retry/rate-limit/audit) → channel plugins**; SQLite as the cross-cutting store; REST + SSE + embedded Web UI as the egress.

![MiBeeBell architecture: ingest → event bus → pipeline → dispatcher → channels, SQLite cross-cutting](images/architecture-pipeline.svg)

## Three key invariants

1. **Publishers never block**: ingest callbacks only normalize and enqueue without blocking; a full bus drops the oldest and counts (`bell_events_dropped_total{stage=bus}`) — **the API never 5xxs because of downstream trouble**.
2. **Recoveries pass immediately**: a `resolved` event paired by `dedup_key` skips dedup/cooldown (silence still applies) and cancels the pending escalation timer.
3. **A pipeline failure ≠ a lost event**: any stage panic is recovered; the event degrades to "no-pipeline mode" straight to the default channel group, with an audit event.

## Backpressure model

| Queue | Default capacity | When full |
|---|---|---|
| eventbus (ingest→pipeline) | 1024 | drop-oldest + counter |
| dispatch queue (pipeline→channel) | 256 per channel | drop-oldest + counter |
| escalation timers | persisted table | never lost (the only persisted queue; timers survive restarts) |

SQLite: WAL mode, `busy_timeout=5s`, a single write connection (serialized writes), multiple readers. Comfortable from hundreds to tens of thousands of events per day.

## Migratable storage

Repositories depend only on `database/sql`; SQL stays ANSI (unix-millisecond integers, JSON in TEXT with app-side codecs, `RETURNING id`, `?` placeholders rebound per dialect) — SQLite dialect specifics live only inside the connector. The `storage.driver` slot reserves PostgreSQL; migrating = implementing a pg connector + the postgres migration dir, with zero repository or pipeline changes (`migrations/postgres/` is maintained in the same PRs as sqlite).

## The config / runtime-state boundary

```
config.yaml              → bootstrap: server / auth / storage / ingest connections
SQLite rules table       → routing rules (UI-editable)
SQLite channels table    → channel configs (encrypted credentials, UI-editable)
config.yaml rules/channels → first-boot seed only (imported once when DB is empty); DB wins afterwards
mibee-bell export        → reverse-export YAML (backup/migration)
```

This boundary means rule/channel changes never touch config files, and config.yaml stays safe for version control (bootstrap and credentials separated).

## Data flow at a glance

![MiBeeBell data flow: ingest → bus → pipeline → render → dispatch → channels, mesh and AI cross-cutting](images/architecture-dataflow.svg)

## Observability

- `/metrics` (Prometheus): `bell_events_ingested_total{source}`, `bell_notifications_total{channel,status}`, `bell_events_dropped_total{stage}`, `bell_channel_health{channel}`, `bell_escalation_pending`, `bell_pipeline_stage_duration_seconds`, `bell_ai_calls_total{analyzer,result}`, `bell_enrich_total`.
- Structured logs (JSON optional): drops, escalations, silence hits and channel degradation all carry event_id.

## Deployment shape

A zero-dependency single binary (linux/amd64 · arm64 · armv6, windows/amd64) plus a native fnOS app. Install steps and the systemd unit example live in [Deployment](deployment.md). Default port `:9080`.
