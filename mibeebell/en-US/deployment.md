# Deployment

One binary (`CGO_ENABLED=0`) for **linux/arm64 · armv7 · amd64** and **windows/amd64** — the same artifact from a Raspberry Pi 3B to an x86 server. Default port `:9080`.

## Install options

| Option | Good for | Notes |
|---|---|---|
| Source build | any platform | `git clone` + `make build`; the web UI ships pre-built and embedded, no node needed |
| systemd | Linux daemons | `deploy/mibee-bell.service` + `deploy/install.sh` (/opt/mibee-bell layout) |
| Docker | NAS / containers | `deploy/compose.cn.yml` (CN topology) / `compose.overseas.yml` (overseas) |
| fnOS app | fnOS NAS | `deploy/fnos/` packaging (fnOS desktop integration, gateway SSO) |
| Windows | desktop/server | cross-compile `GOOS=windows GOARCH=amd64`, run `mibee-bell.exe -config config.yaml` |

## systemd (recommended)

```bash
# 1) build the linux binary locally
make build

# 2) install to /opt/mibee-bell + unit + system user "mibee"
sudo deploy/install.sh ./mibee-bell

# 3) write the config (first time)
sudo $EDITOR /opt/mibee-bell/config.yaml
sudo /opt/mibee-bell/mibee-bell hash-password   # → auth.password_hash
sudo /opt/mibee-bell/mibee-bell encrypt-config  # → credential-encryption crypto.key (optional)

# 4) go
sudo systemctl enable --now mibee-bell
journalctl -u mibee-bell -f
```

The unit ships hardened: `ProtectSystem=strict`, `ReadWritePaths=/opt/mibee-bell`, `PrivateTmp`. Upgrades = swap the binary (the old one is kept as a timestamped `.bak`) + `systemctl restart`.

## Docker Compose

```bash
cd deploy
docker compose -f compose.cn.yml up -d
```

The CN example bakes in the local pattern: Telegram via a per-channel egress proxy; the overseas topology has all channels direct. Volumes mount SQLite and config.yaml.

## Raspberry Pi / low-spec devices

- Measured single-instance RSS ≈ 15–21MB; the design line is < 80MB on a Pi 3B (512MB).
- Cross-compile: `CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build ./cmd/mibee-bell`.
- The mesh naturally fits a "home server + Pi + VPS" spread (see [High Availability](mesh.md)).

## Data & backups

- SQLite single file (default `./mibee-bell.db`, WAL). Backup: cold copy on stop, or `sqlite3 mibee-bell.db ".backup ..."`.
- Retention (daily cleanup): events 30d / notifications 90d / deliveries 30d / feedback forever (tunable via `storage.retention`).
- Migration/disaster recovery: `mibee-bell export` reverse-exports rules and channels as YAML.

## Reverse proxy

UI and API share one port — proxy `:9080` directly; SSE (`/api/stream`) needs buffering off (nginx: `proxy_buffering off`). Ingest endpoints can stay LAN-only as needed.

## Monitoring Bell itself

- `/metrics` (Prometheus) + the `deploy/grafana-bell.json` dashboard.
- Key metrics: `bell_events_dropped_total{stage}` (backpressure drops), `bell_channel_health{channel}` (degraded channels), `bell_escalation_pending` (armed escalations).
- In multi-node deployments the node_down dead-man switch covers "Bell itself died" automatically.
