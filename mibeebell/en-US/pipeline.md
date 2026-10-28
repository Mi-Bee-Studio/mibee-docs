# Alert Pipeline (4 Gates + L0)

The pipeline is MiBeeBell's core differentiator: **ntfy/Gotify-class tools forward every event as-is; MiBeeBell puts a configurable pipeline before the notification exit** — dedup, aggregation, cooldown, escalation, silence, plus an always-on pure-algorithm denoise layer (L0). Design principles: pure functions with explicit state, every gate table-testable, every parameter overridable per rule.

## Processing order

```
event → [silence] → [persist_filter N-points] → [dedup + similarity aggregation]
      → [cooldown] → [EWMA frequency tagging] → [common-cause merge] → [dependency suppression]
      → rule routing recheck → i18n render → dispatch (escalation timer armed)
```

The four gates and L0 algorithms all implement the same `pkg/bellapi.Processor` interface on one chain — **adding an algorithm plugs in a new Processor; the core never changes**.

## Gate 1: silence

Events inside a maintenance window are stored and marked `silenced`; no notification is produced.

- **Matcher**: `match` (source/type/tags subset matching, same syntax as rules) + a time window — absolute (`starts_at ~ ends_at`) or **daily recurring** (e.g. `23:00-06:00`, crossing midnight supported).
- **Creation**: REST API (including one-tap silence by `dedup_key`), the Web UI silence buttons, Scout notification actions.
- **Recoveries are exempt** — recoveries must stay visible, or the end of a silence window produces a wave of fake alerts.

## Gate 2: dedup (aggregation)

Events sharing a `dedup_key` within the window merge into **one** notification: the first fires immediately (low latency); repeats bump `events.dedup_count` (the UI can show "this camera dropped 12 times in the last hour").

- Window defaults to 60s, rule-overridable (`dedup_seconds`).
- **Similarity aggregation (simagg)**: textually similar events merge within the window — "30 motion events in 30 seconds collapse to 1 notification" is a v0.1 acceptance test.

## Gate 3: cooldown

A per-`(rule × source_ref)` anti-flap window: **recurrences after a notification has fired** only update the counter within the window (dedup merges a burst; cooldown throttles re-notification). Default 5 minutes; `last_triggered_at` is persisted, so cooldowns survive restarts. Recoveries are never cooled down.

## Gate 4: escalation

After an `error`-level notification fires, if no paired `resolved` arrives within `escalate_after` (default 15 minutes):

1. severity is raised to `critical` and the notification is marked "escalated";
2. the `escalate_to` channel group is notified additionally (paid channels are only ever allowed here);
3. at most one hop.

Timers persist in the `escalations` table — **they survive process restarts**; a paired resolve cancels them.

## L0 algorithm layer (pure Go, always on)

"Dynamic denoising" that depends on no model and no network; all parameters rule-overridable:

| Algorithm | What it solves | Defaults |
|---|---|---|
| **N-consecutive-points filter** | glitch false positives: a single sample blip notifies nothing | N=3, only for warning and below; error/critical never filtered |
| **EWMA dynamic baseline** | frequency spikes (50 drops in one night = a network problem, not 50 incidents) | alpha=0.3, 24h window |
| **Similarity aggregation (simagg)** | 30 same-source events in 30s → 1 | 60s window, same source_ref only |
| **Common-cause merge** | one root cause (switch loses power) fanning out 30 device alerts → 1 "zone event" | 60s window, merge key = `tags.zone` or shared `tags.upstream` |
| **Dependency suppression** | NVR offline suppresses all derived camera alerts | explicit `tags.depends_on` topology |
| **Token bucket + daily budget** | channel rate limits + paid-channel bill protection | channels self-report limits; `daily_budget` mandatory for paid channels |

## Degradation & self-healing

- Any stage panic → recover → the event goes out in "no-pipeline mode" to the default channel group + a `pipeline_degraded` audit event.
- AI (L1/L2) hangs off the enrich hook: 2s timeout / budget exhaustion / call failure → fall straight back to L0 templates, **the event still goes out**.
- AI only downgrades severity (the original value is kept in the `ai_severity_from` tag) and can never drop an event — see [AI Enhancements](ai.md).

## A worked example

Feeding 18 events with the same `dedup_key` into a demo instance produces this server log:

```
level=INFO msg="event dropped by gate" gate=dedup reason="dedup ×2"
...
level=INFO msg="event dropped by gate" gate=dedup reason="dedup ×18"
level=INFO msg="escalation scheduled" dedup_key=mibee-nvr/connection/cam-front
level=INFO msg="escalations cancelled by resolve" dedup_key=mibee-nvr/connection/cam-front count=1
```

18 events → 1 notification (first fires immediately); the 15-minute escalation timer arms; the recovery event cancels it automatically.
