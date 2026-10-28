# Quick Start

Goal: get MiBeeBell running on a Linux box, fire your first alert and receive the notification — **in about 5 minutes**.

> Commands are copy-paste ready. Example addresses use the documentation range `192.0.2.10` — replace with your instance address.

## 1. Build & install

Build from source (the web UI ships pre-built and embedded — no node needed):

```bash
git clone https://github.com/Mi-Bee-Studio/MiBeeBell.git
cd MiBeeBell
make build          # produces ./mibee-bell (CGO_ENABLED=0, cross-compile friendly)
```

System install (optional, systemd-managed `/opt/mibee-bell` layout):

```bash
sudo deploy/install.sh ./mibee-bell
```

## 2. Write a minimal config

```bash
./mibee-bell hash-password --password 'your-admin-password'
# prints {"password_hash":"$2a$10$..."} — paste it below

cat > config.yaml <<'EOF'
server:
  listen: ":9080"

auth:
  password_hash: "$2a$10$..."        # output of the previous step

storage:
  path: ./mibee-bell.db

ingest:
  webhook_tokens:
    - token: "my-ingest-token-1"      # any long random string; stored on both sides
      default_source: "myapp"
  alertmanager: true                  # also enable the Prometheus/Grafana entry
EOF
```

> With an empty `auth.password_hash`, the admin API and Web UI stay disabled (logged warning); ingest endpoints are unaffected.

## 3. Start

```bash
./mibee-bell -config config.yaml
# log: mibee-bell listening addr=:9080 ... api=true
curl http://127.0.0.1:9080/healthz
# {"status":"ok","service":"mibee-bell",...}
```

Open `http://192.0.2.10:9080` in a browser, log in with the admin password — you land in the notification center.

## 4. Add a channel

UI left nav "Channels" → New. The quickest channel to verify with is a **generic Webhook** (any URL that accepts POST); for real use, your group bot:

| Channel | Key credential |
|---|---|
| WeCom robot | the `key` from the group-bot webhook URL |
| DingTalk robot | `access_token` + signing `secret` |
| Feishu robot | webhook URL (optional signature) |
| Telegram | bot `token` + `chat_id` (per-channel proxy supported) |
| SMTP | host / account / password |
| Generic Webhook | target URL (optional HMAC-SHA256 signing) |

Hit "Send test" to verify. Credentials are AES-256-GCM encrypted at rest; the UI and API only ever show `***`.

## 5. Create a rule

UI "Rules" → New:

```yaml
name: myapp-errors
match:
  source: myapp            # matches the source bound to your ingest token
  severity_gte: error
channels: [your-channel]
cooldown: 5m               # no re-bombardment of the same subject within 5 minutes
escalate_after: 15m        # not resolved in 15m → escalate & resend
```

## 6. Fire your first alert

```bash
curl -X POST "http://192.0.2.10:9080/api/ingest/webhook/my-ingest-token-1" \
  -H "Content-Type: application/json" \
  -d '{"type":"demo_alert","severity":"error","title":"first test alert",
       "body":"hello from curl","source_ref":"demo-1"}'
# 202 {"event_id":"..."} — accepted into the pipeline; check the UI for the notification & deliveries
```

Send a **recovery event** with the same `dedup_key` and watch the paired "resolved" notice arrive:

```bash
curl -X POST "http://192.0.2.10:9080/api/ingest/webhook/my-ingest-token-1" \
  -H "Content-Type: application/json" \
  -d '{"type":"demo_alert","severity":"info","title":"first test alert resolved",
       "source_ref":"demo-1","resolved":true}'
```

## 7. Wire up existing systems (zero refactor)

```yaml
# Prometheus Alertmanager — alertmanager.yml
route:
  receivers: [bell]
receivers:
  - name: bell
    webhook_configs:
      - url: http://192.0.2.10:9080/api/ingest/alertmanager
```

Existing Bark-ecosystem scripts just swap `api.day.app/yourkey` for `http://192.0.2.10:9080/yourkey`. More entries in [Event Ingest](ingest.md).

## Requirements & next steps

- Any Linux (arm64 / armv7 / amd64) or Windows x64; 512MB RAM is plenty.
- Next: [Web UI Tour](web-ui.md) · [Alert Pipeline](pipeline.md) · [High Availability](mesh.md)
