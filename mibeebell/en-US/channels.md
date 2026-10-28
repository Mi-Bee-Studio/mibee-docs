# Notification Channels

Principle (iron law 3): **one channel = one file = one interface**. A platform policy change kills at most that adapter. Adapters do nothing but "rendered message → that platform's API"; retry, rate limiting, audit and health probing are centralized in the dispatcher.

## Built-in channels

| type | Channel | Declared rate limit (default) | Notes |
|---|---|---|---|
| `wecom_robot` | WeCom group robot | 18 / 60s | official 20/min, safety margin |
| `dingtalk_robot` | DingTalk custom robot | 18 / 60s | signed (secret + timestamp) |
| `feishu_robot` | Feishu custom robot | 18 / 60s | optional signature verification |
| `smtp` | SMTP email | 6 / 60s | QQ/163/SES/Resend all work; body Markdown→HTML |
| `telegram` | Telegram bot | 20 / 60s | per-channel egress proxy (socks5/http) |
| `slack` | Slack incoming webhook | 1 / 1s | |
| `discord` | Discord webhook | 5 / 60s | official 50/min |
| `webhook` | generic webhook | 10 / 60s | JSON POST + optional HMAC-SHA256 signature header |
| `scout` | MiBeeScout notification center | — | events stored + consumed via the notification-center API/SSE |

Planned for v0.3: `bark` (native client push), `ntfy`, `pushover`, `wecom_app`, and the paid group (`twilio` / `pagerduty` / `opsgenie` — allowed only in `escalate_to` chains). Long-tail channels are covered by the generic adapter layer.

## Channel configuration

Configured on the UI "Channels" page or via `POST /api/v1/channels`, each channel independently:

- **Credential fields** (key/token/secret/webhook url) — AES-256-GCM encrypted at rest (`enc:` prefix), decrypted just-in-time for the adapter, always masked `***` in UI/API output.
- **`proxy`**: per-channel egress proxy — e.g. Telegram via a local socks5 on a CN instance, without affecting other channels.
- **`timeout`**: default 10s.
- **Enable/disable & health**: 5 consecutive delivery failures → `health_status=degraded` → UI marker + optional "time to switch channels" admin notice. Keep ≥2 channels configured at all times.

```yaml
# config.yaml first-boot seed example (DB wins afterwards; UI-editable)
channels:
  wecom-ops: { type: wecom_robot, key: "enc:..." }
  tg:        { type: telegram, token: "enc:...", chat_id: "123", proxy: "socks5://127.0.0.1:1080" }
```

## Dispatcher responsibilities (what adapters don't do)

1. **Retry**: exponential backoff 1s/2s/4s/8s/16s, at most 5 attempts; 4xx (except 429) is not retried — retrying a config error is pointless.
2. **Rate limiting**: per-channel token bucket (min of the channel's declared limit and config) — storms queue up without blocking.
3. **Audit**: every attempt lands in `deliveries` (status / attempt / response code / error digest), visible in the UI.
4. **Daily budget**: paid channels circuit-break on `daily_budget_events` — over budget means skipped + audited, never blocking other channels.

## Message rendering

- Template matrix: `(severity × channel_family × lang)` with zh-CN / en-US built in, falling back to en-US.
- Family differences: IM channels use their own Markdown dialects (WeCom markdown / DingTalk markdown / Feishu post / TG HTML); SMTP uses HTML; the generic webhook emits structured JSON with all event fields.
- Every notification carries: severity icon + title, body, snapshot thumbnail (when `snapshot_url` exists and the channel supports it), an RFC 3339 timestamp (local timezone), and a source footer (MiBeeBell + rule name).
- Aggregation counts (×N) and "escalated" markers are injected at render time.

## Adding a channel (contributors)

1. Create `internal/channel/<name>/`, implement the `Channel` interface (`Name` / `Send` / `RateLimit`) + self-register in `init()`;
2. Enable with a DB or config entry — **no core changes**;
3. Unit tests, three cases against an httptest mock (success / rate-limited / 4xx);
4. Add a row to the docs table (declared rate limit + credential fields).

The interfaces live in `pkg/bellapi` — a stable external contract; third-party channels can live in their own repositories.
