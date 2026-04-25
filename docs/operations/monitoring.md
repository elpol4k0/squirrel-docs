# Monitoring

squirrel exposes Prometheus metrics when running in daemon mode. Use these to track backup health, durations, and errors.

``` mermaid
flowchart LR
    SQ[squirrel daemon\n--metrics :9090] -- scrape --> PROM[Prometheus]
    PROM --> GRAFANA[Grafana\ndashboards]
    PROM --> ALERT[Alertmanager\nalerts]
    SQ -- webhook --> HC[healthchecks.io\nUptime Kuma]
```

---

## Enabling Metrics

Pass `--metrics <addr>` to `squirrel daemon`:

```bash
squirrel daemon --metrics :9090
```

Metrics are available at:

```
http://host:9090/metrics
```

---

## Exposed Metrics

| Metric | Type | Description |
|---|---|---|
| `squirrel_backup_duration_seconds` | Histogram | Backup job duration in seconds, labeled by profile |
| `squirrel_backup_bytes_processed` | Counter | Total bytes processed across all backups |
| `squirrel_backup_errors_total` | Counter | Total number of backup errors, labeled by profile |
| `squirrel_last_success_timestamp` | Gauge | Unix timestamp of the last successful backup, labeled by profile |

Standard Go runtime metrics (`go_goroutines`, `go_memstats_*`, etc.) are also exposed.

---

## Prometheus Configuration

```yaml
# prometheus.yml
scrape_configs:
  - job_name: squirrel
    static_configs:
      - targets: ['squirrel-host:9090']
    scrape_interval: 60s
```

---

## Alerting Rules

### Backup Not Running

```yaml
groups:
  - name: squirrel
    rules:
      - alert: SquirrelBackupNotRunning
        expr: time() - squirrel_last_success_timestamp{profile="pg-prod"} > 86400
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "squirrel backup {{ $labels.profile }} has not succeeded in 24h"
```

### Backup Errors

```yaml
      - alert: SquirrelBackupErrors
        expr: increase(squirrel_backup_errors_total[1h]) > 0
        labels:
          severity: critical
        annotations:
          summary: "squirrel backup errors detected for {{ $labels.profile }}"
```

---

## Grafana Dashboard

Import a dashboard using the following panel queries:

**Backup duration (last 7 days):**
```promql
rate(squirrel_backup_duration_seconds_sum[24h])
  / rate(squirrel_backup_duration_seconds_count[24h])
```

**Time since last success per profile:**
```promql
time() - squirrel_last_success_timestamp
```

**Error rate:**
```promql
rate(squirrel_backup_errors_total[1h])
```

---

## Healthcheck Webhooks

As an alternative to Prometheus, use healthcheck webhooks in hooks to integrate with services like [healthchecks.io](https://healthchecks.io) or Uptime Kuma:

```yaml
hooks:
  post-success:
    - webhook: https://hc-ping.com/your-uuid-here
  post-failure:
    - webhook: https://hc-ping.com/your-uuid-here/fail
```

healthchecks.io will alert you if the ping is not received within the expected window.

---

## Log Output

squirrel uses structured logging (`log/slog`) with the following key fields:

| Field | Description |
|---|---|
| `level` | `INFO`, `WARN`, `ERROR`, `DEBUG` |
| `msg` | Log message |
| `profile` | Profile name (in daemon mode) |
| `blobs` | Blob count |
| `bytes` | Byte count |
| `startLSN` | PostgreSQL WAL position |
| `segments` | WAL/binlog segment count |

Example log output:

```
2026-04-25 02:00:01 INFO backup started profile=pg-prod
2026-04-25 02:00:03 INFO base backup started startLSN=0/5000000 tablespaces=0
2026-04-25 02:00:45 INFO tablespace streamed label=base bytes=2147483648 blobs=512
2026-04-25 02:00:46 INFO base backup finished endLSN=0/6000000
2026-04-25 02:00:46 INFO WAL streaming started slot=squirrel startLSN=0/5000000
2026-04-25 02:00:56 INFO WAL segment flushed name=000000010000000000000005 bytes=16777216
```
