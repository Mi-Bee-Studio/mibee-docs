# Deployment

A zero-dependency single binary for **linux/amd64 · arm64 · armv6** and **windows/amd64**, plus a **native fnOS app** (arm64 / x86) — the same artifact from a Raspberry Pi 3B to an x86 server. Default port `:9080`.

## Install options

| Option | Good for | Notes |
|---|---|---|
| Release download | any platform | grab tar.gz/zip from the [release hub](https://github.com/mickeyzzc/releases/releases/tag/mibeebell-v0.1.0), verify with `checksums.txt` |
| fnOS app | fnOS NAS | download the `.fpk` (arm64 / x86) → App Center manual install; desktop SSO, gateway login-free |
| systemd | Linux daemons | downloaded binary + the unit example below (/opt/mibee-bell layout) |
| Windows | desktop/server | unzip `mibee-bell_windows_amd64.zip`, run `mibee-bell.exe -config config.yaml` |
| Docker | containers | planned (v1.0 full distribution channels); use the binary for now |

## systemd (recommended)

```bash
# 1) download and unpack the linux binary
tar xzf mibee-bell_linux_amd64.tar.gz

# 2) install to /opt/mibee-bell + system user "mibee"
sudo useradd -r -s /usr/sbin/nologin mibee || true
sudo mkdir -p /opt/mibee-bell && sudo mv mibee-bell /opt/mibee-bell/

# 3) write the config (first time)
sudo $EDITOR /opt/mibee-bell/config.yaml
sudo /opt/mibee-bell/mibee-bell hash-password   # → auth.password_hash
sudo /opt/mibee-bell/mibee-bell encrypt-config  # → credential-encryption crypto.key (optional)

# 4) systemd unit (/etc/systemd/system/mibee-bell.service)
sudo tee /etc/systemd/system/mibee-bell.service <<'EOF'
[Unit]
Description=MiBeeBell alert notification hub
After=network-online.target

[Service]
User=mibee
WorkingDirectory=/opt/mibee-bell
ExecStart=/opt/mibee-bell/mibee-bell -config /opt/mibee-bell/config.yaml
Restart=always
RestartSec=3
# hardening
ProtectSystem=strict
ReadWritePaths=/opt/mibee-bell
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

# 5) go
sudo systemctl daemon-reload && sudo systemctl enable --now mibee-bell
journalctl -u mibee-bell -f
```

Upgrades = download the new binary and swap it (the old one is kept as a timestamped `.bak`) + `systemctl restart`.

## Raspberry Pi / low-spec devices

- Measured single-instance RSS ≈ 15–21MB; the design line is < 80MB on a Pi 3B (512MB).
- Download `mibee-bell_linux_arm64.tar.gz` (Raspberry Pi class) or `armv6` (older 32-bit boards).
- The mesh naturally fits a "home server + Pi + VPS" spread (see [High Availability](mesh.md)).

## Data & backups

- SQLite single file (default `./mibee-bell.db`, WAL). Backup: cold copy on stop, or `sqlite3 mibee-bell.db ".backup ..."`.
- Retention (daily cleanup): events 30d / notifications 90d / deliveries 30d / feedback forever (tunable via `storage.retention`).
- Migration/disaster recovery: `mibee-bell export` reverse-exports rules and channels as YAML.

## Reverse proxy

UI and API share one port — proxy `:9080` directly; SSE (`/api/stream`) needs buffering off (nginx: `proxy_buffering off`). Ingest endpoints can stay LAN-only as needed.

## Monitoring Bell itself

- `/metrics` (Prometheus) — build your own Grafana panel on the metrics below.
- Key metrics: `bell_events_dropped_total{stage}` (backpressure drops), `bell_channel_health{channel}` (degraded channels), `bell_escalation_pending` (armed escalations).
- In multi-node deployments the node_down dead-man switch covers "Bell itself died" automatically.
