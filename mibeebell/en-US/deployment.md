# Deployment

One binary (`CGO_ENABLED=0`) for **linux/arm64 · armv7 · amd64** and **windows/amd64** — the same artifact from a Raspberry Pi 3B to an x86 server. Default port `:9080`.

## Install options

| Option | Good for | Notes |
|---|---|---|
| **[Releases hub download](https://github.com/mickeyzzc/releases/releases/tag/mibeebell-v0.1.0)** | any platform (recommended to start) | since the first release v0.1.0 (2026-11-04): Linux/Windows single binaries and the fnOS fpk package, no build step |
| systemd | Linux daemons | `deploy/mibee-bell.service` + `deploy/install.sh` (/opt/mibee-bell layout) |
| Docker | NAS / containers | `deploy/compose.cn.yml` (CN topology) / `compose.overseas.yml` (overseas) |
| fnOS app | fnOS NAS | manual fpk install via App Center (fnOS desktop integration, gateway SSO) |
| Windows | desktop/server | `mibee-bell` windows/amd64 build from Releases, run `mibee-bell.exe -config config.yaml` |

After installing, the first thing to do is open the Web home and **set the admin password** (see [Quick Start](quick-start.md) §2); everything else is configured in the Web settings and written back to yaml automatically.

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

## Upgrades

- **fnOS**: upgrade directly in App Center — the data directory is preserved.
- **Binary/systemd**: swap the binary (the install.sh path keeps the old one as a timestamped `.bak`) → `systemctl restart`; database migrations run automatically at startup (schema is add-only). Back up as usual before upgrading.
- **Docker**: `docker compose pull && up -d`; data volumes untouched = data preserved.
- Full steps, rollback and cross-machine migration: `docs/upgrade.md` in the repository.

## Data & backups

- SQLite single file (default `./mibee-bell.db`, WAL). Backup: cold copy on stop, or `sqlite3 mibee-bell.db ".backup ..."` (copy config.yaml together with the db).
- Retention (daily cleanup): events 30d / notifications 90d / deliveries 30d (Settings → System & retention, "0" = forever).
- Config export: `GET /api/v1/rules` and `GET /api/v1/channels` (credentials masked `***`) for JSON snapshots.

## Reverse proxy

UI and API share one port — proxy `:9080` directly; SSE (`/api/stream`) needs buffering off (nginx: `proxy_buffering off`). Ingest endpoints can stay LAN-only as needed.

## Monitoring Bell itself

- `/metrics` (Prometheus) + the `deploy/grafana-bell.json` dashboard.
- Key metrics: `bell_events_dropped_total{stage}` (backpressure drops), `bell_channel_health{channel}` (degraded channels), `bell_escalation_pending` (armed escalations).
- In multi-node deployments the node_down dead-man switch covers "Bell itself died" automatically.
