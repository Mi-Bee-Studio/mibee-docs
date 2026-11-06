# FAQ

## Positioning & capabilities

**Q: How is MiBeeHive different from a cloud drive or a download manager?**
Different job: cloud drives store files for people; MiBeeHive supplies a LAN — OS installs (PXE + unattended), restocking (public releases land locally), sharing (WebDAV) and inspection (jobs / keys / backups / monitoring) in one place, with all data on your own hardware.

**Q: What hardware does it run on?**
ARM64 devices first (NAS / Raspberry Pi class, ≥1GB RAM); x86_64 works too. The acceptance line is "a low-spec ARM64 NAS runs it comfortably".

**Q: Should I expose it to the internet?**
No. MiBeeHive is a LAN supply hub; if you need remote access, build it yourself with a VPN / tunnel and proper access control.

## First use

**Q: How do I log in the first time?**
Log in to the web UI with the initial admin account and **change the default password immediately** (Settings page). The admin password is stored hashed only.

**Q: Where do foraged files, images and backups live?**
All under the data directory configured at deploy time (per-module subdirectories); a single SQLite file holds metadata. The storage page in the UI shows usage.

## Daily operations

**Q: How do I back up?**
The ops center offers one-click full backup (database + config + scripts); archives can be downloaded from the UI. Always back up before upgrading or migrating.

**Q: How do I upgrade?**
Swap the binary and restart the service — the database migrates automatically. Back up first. Release notes land on this docs site as versions ship.

**Q: I lost a key / a key leaked — now what?**
Keys are write-only (plaintext never re-displayed) — rotate the value in key management; the old value dies instantly. Re-distribute the new value to your scripts.

**Q: How do I hook up monitoring?**
`/metrics` serves Prometheus; the system status page configures disk-threshold alerts. Pair with node_exporter for full-machine resources.

## Release & availability

**Q: Where can I download MiBeeHive?**
It is not publicly released yet — the distribution channel is to be announced. This manual is published ahead of availability.

**Q: How do I report a security issue?**
Please don't disclose details publicly — reach out privately through the official site contact; we'll respond promptly.
