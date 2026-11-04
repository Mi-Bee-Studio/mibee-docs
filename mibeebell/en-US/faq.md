# FAQ & Troubleshooting

Quick self-check: `curl -s http://<bell>:9080/healthz` (liveness + version) — if that fails, look at the service process and port first; if it works, find your symptom below.

## Install & login

**Q: A brand-new instance shows a login wall and won't let me set a password?**
v0.1.0–v0.1.2 have a known defect: the first-run password page never appears (a GET request carrying an empty body gets rejected by the browser). Upgrade to a version containing the fix; on older versions the CLI workaround is `./mibee-bell hash-password` — paste the hash into config.yaml's `auth.password_hash` and restart.

**Q: Forgot the admin password?**
On the server run `./mibee-bell hash-password`, enter a new password at the prompt, paste the printed bcrypt hash into config.yaml's `auth.password_hash` and restart.

**Q: The password is right but login keeps failing?**
Five consecutive failures from one client trigger a 15-minute lockout (even the correct password is rejected) — usually an auto-retry script hammering after a password change. Stop the retries and wait out the lockout.

**Q: The fnOS desktop works, why does direct port access ask for a password?**
The fnOS desktop goes through gateway single-sign-on; direct access to `:9080` is a separate auth path that uses the local admin password.

## Notification pipeline

**Q: The sender got a 202 but nothing shows in the notification center?**
①Make sure the token wasn't deleted (deletion takes effect immediately; further sends get 404); ②check a matching rule exists and is enabled — an event's `source` comes from the ingest token's bound default source, not your payload; ③warning-and-below events go through the N-points filter: the same dedup key needs ≥3 occurrences within 10 minutes to notify (error/critical bypass) — set the rule's persist-filter to 1 for low-frequency probe sources.

**Q: The same alert keeps bombing me?**
Inconsistent `source_ref` on the sender side produces different dedup keys — fix `source_ref` or send an explicit `dedup_key`; also consider raising the rule's cooldown seconds.

**Q: Recovery events don't pair with incidents?**
Recoveries must share the incident's **dedup key** (default `source/type/source_ref`), otherwise they're treated as standalone events.

**Q: A channel keeps failing delivery?**
Hit "Send test" on the channel page to reproduce instantly and read the error: expired webhook URLs/credentials, unreachable egress network, channel rate limits, and wrong proxy settings are the four big root causes; the delivery details (expand any notification) carry per-attempt status codes and error summaries.

**Q: I don't want to be buzzed at night?**
Two ways: one-tap silence per dedup key (30 minutes – 24 hours) from any notification; or a silence-window rule `{"cron":"23:00-06:00"}` as a daily window (crossing midnight supported).

## AI & advanced

**Q: AI summaries come and go?**
Settings → AI & budget → the AI health card: cloud L2 is protected by a daily budget (default 100 calls/day) and a circuit breaker — while tripped or exhausted, notifications still go out but without summaries; the budget resets each calendar day. The embedded local model (L1) has no budget concept and connects to nothing.

**Q: Can AI drop my alerts?**
Not by default — AI only summarizes/tags/triages; the pipeline-intervention capability that lets AI suppress/aggregate/reorder is fully off by default and requires shadow-mode (record-only) validation before enabling, with automatic fail-open when AI fails.

**Q: The live badge went gray / the flow graph stopped moving?**
SSE is disconnected. The most common cause is a reverse proxy buffering the event stream — disable buffering for `/api/stream` (nginx `proxy_buffering off`). Reconnects resume via `Last-Event-ID`; data older than the 512-entry buffer is a page refresh away.

**Q: Will upgrading lose data?**
No. Config and the SQLite database live in the data directory and upgrades preserve them; database migrations are add-only. Back up as usual before upgrading (copy config.yaml + bell.db) — details in [Deployment](deployment.md).

## Still stuck?

- Repository manuals: [docs/user-guide.md](https://github.com/Mi-Bee-Studio/MiBeeBell/blob/main/docs/user-guide.md) (with the full quick troubleshooting table) and [docs/troubleshooting.md](https://github.com/Mi-Bee-Studio/MiBeeBell/blob/main/docs/troubleshooting.md) (the expanded eight-section runbook);
- When filing a GitHub issue, please include: the version string (`/healthz`), relevant log lines (redacted), and one minimal reproducible event.
