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

Metrics exposed:

- cumulative counters (hash-deduped over a 31d log fetch), for rate()/trend panels:
  `spamd_messages_total{verdict,user}`, `spamd_messages_by_domain_total{verdict,domain}`,
  `spamd_messages_by_sender_total{verdict,sender,domain}`,
  `spamd_message_size_bytes_total{verdict}`, `spamd_scantime_seconds_total{verdict}`,
  `spamd_score_sum{verdict="spam"}` (spam scores are always >= required_score, so this
  stays monotonic), `spamd_bayes_sum{verdict}`, `spamd_result_lines_total`
- windowed gauges `spamd_window_messages{window,verdict,user}`,
  `spamd_window_messages_by_domain{window,verdict,domain}`,
  `spamd_window_messages_by_sender{window,verdict,sender,domain}`,
  `spamd_window_size_bytes{window,verdict}`, `spamd_window_scantime_seconds{window,verdict}`,
  `spamd_window_score_sum{window,verdict}` — recomputed from the raw logs on every poll
  (windows: 1h, 6h, 24h, 7d, 30d; limited by container log rotation)
- health: `spamd_exporter_errors_total{op,pod}`, `spamd_exporter_last_run_timestamp_seconds`

The share/top-N dashboard panels query the **windowed gauges**, not `increase()` on the
counters: a first-time spammer only ever appears as a flat series at value 1, and
`increase()` over a window that starts before the series exists returns 0 for such
series — so increase()-based top-N panels silently hide one-off spammers. The gauges
are also immune to exporter restarts (no state is kept; everything is recomputed from
the logs). The trend panels (% over time, msg/min, bytes/s) use `rate()` on the counters,
which handles counter resets correctly.

The exporter polls the log API every 30s (`POLL_SECONDS`), once per window per spamd pod.

## Dashboard panels

- Pie chart: spam vs ham share over `$window` (1h/6h/24h/7d/30d variable, windowed gauges)
- Stats: spam %, message counts, avg spam score (required_score is 3.0), avg scan time
- Time series: spam/ham % over time, messages/min by verdict, bytes/s scanned (rate on counters)
- Bar gauges: top 10 spamming domains and top 10 spamming senders (Message-ID) over `$window`
- Stat: exporter health (result lines parsed, poll age)

## Operating notes

- Exporter needs `pods` list/get and `pods/log` get in the `mail` namespace (Role in
  `apps/07-mail/spamd-exporter/rbac.yaml`).
- The ServiceMonitor carries `release: prometheus-release` because the Prometheus CR
  selects ServiceMonitors by that label in all namespaces.
- Dashboard edits: change the JSON in `apps/01-monitoring/config/spamd-dashboard.yaml`
  (the `spamd.json` block) — the sidecar reloads it within ~30s.
