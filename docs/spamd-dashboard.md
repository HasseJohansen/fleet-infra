# spamd Grafana dashboard

Grafana dashboard **"spamd — Spam vs Ham"** (uid `spamd-overview`, General folder) fed by
Prometheus metrics that are parsed out of spamd's pod logs.

There is no Loki in this cluster, so logs are turned into metrics instead:

```
spamd pods (--syslog=stderr) → container runtime → K8s pod log API
  → spamd-exporter (apps/07-mail/spamd-exporter, python:3.13-alpine, stdlib only)
  → /metrics on :8080 → ServiceMonitor (label release=prometheus-release)
  → Prometheus → Grafana dashboard (apps/01-monitoring/config/spamd-dashboard.yaml,
    ConfigMap grafana-dashboard-spamd, label grafana_dashboard="1", picked up by the
    grafana-sc-dashboard sidecar into /tmp/dashboards)
```

## What the exporter parses

spamd logs one line per scanned message, e.g.:

```
Nov 23 10:42:53.946 [11] info: spamd: result: Y  7 - BAYES_60,... scantime=0.9,size=139714,user=hhj,uid=2022,required_score=3.0,...,mid=<332146044432314166325010845424033286155160285137@zoo-friends.help>,bayes=0.755469,...
```

The verdict char after `result:` is `Y` = spam, `.`/`N` = ham. Note that spamd does
**not** log the envelope sender or the From header — only the Message-ID — so in the
dashboard "sender" = full Message-ID and "domain" = its domain part. Spammers rotate
the Message-ID local part, so the *domain* panels are usually the useful ones.

Metrics exposed (all counters except the gauge):

- `spamd_messages_total{verdict,user}`
- `spamd_messages_by_domain_total{verdict,domain}`
- `spamd_messages_by_sender_total{verdict,sender,domain}`
- `spamd_message_size_bytes_total{verdict}`
- `spamd_scantime_seconds_total{verdict}`
- `spamd_score_sum{verdict="spam"}` (spam scores are always >= required_score, so this stays monotonic)
- `spamd_bayes_sum{verdict}`
- `spamd_result_lines_total`, `spamd_exporter_errors_total{op,pod}`, `spamd_exporter_last_run_timestamp_seconds`

The exporter polls the log API every 15s (`SINCE_SECONDS=90`, dedup by line hash), and
on (re)start backfills `BACKFILL_SECONDS=86400` once so the counters reflect the last 24h.
Because the counters are in-memory, `increase()` over windows larger than the exporter's
uptime undercounts the backfilled part — after 24h of uptime all `$window` panels are exact.

## Dashboard panels

- Pie chart: spam vs ham share over `$window` (1h/6h/24h/7d/30d variable)
- Stats: spam %, message counts, avg spam score (required_score is 3.0), avg scan time
- Time series: spam/ham % over time, messages/min by verdict, bytes/s scanned
- Bar gauges: top 10 spamming domains and top 10 spamming senders (Message-ID) over `$window`
- Stat: exporter health (result lines parsed, poll age)

## Operating notes

- Exporter needs `pods` list/get and `pods/log` get in the `mail` namespace (Role in
  `apps/07-mail/spamd-exporter/rbac.yaml`).
- The ServiceMonitor carries `release: prometheus-release` because the Prometheus CR
  selects ServiceMonitors by that label in all namespaces.
- Dashboard edits: change the JSON in `apps/01-monitoring/config/spamd-dashboard.yaml`
  (the `spamd.json` block) — the sidecar reloads it within ~30s.
