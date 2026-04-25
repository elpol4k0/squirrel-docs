# squirrel daemon

Run squirrel as a long-running background process with a built-in cron scheduler. Suitable for Docker/container deployments where systemd is unavailable.

```
squirrel daemon [flags]
```

---

## Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--config` | string | platform default | Path to config file. |
| `--profile` | []string | all scheduled | Profiles to activate. If omitted, all profiles with a `schedule:` field are scheduled. |
| `--metrics` | string | — | Address to expose Prometheus metrics on (e.g. `:9090`). If omitted, metrics are disabled. |

---

## How It Works

1. Reads the config file and builds a schedule for each profile that has a `schedule:` field.
2. Runs until a `SIGTERM` or `SIGINT` is received.
3. On `SIGHUP`, reloads the config file without restarting (live reload).
4. Executes `squirrel run <profile>` for each profile at the scheduled time.
5. Exposes Prometheus metrics if `--metrics` is set.

---

## Examples

### Run all scheduled profiles

```bash
squirrel daemon
```

### Run specific profiles only

```bash
squirrel daemon --profile pg-daily --profile files-home
```

### With Prometheus metrics

```bash
squirrel daemon --metrics :9090
```

### Docker Compose

```yaml
services:
  squirrel:
    image: ghcr.io/elpol4k0/squirrel:latest
    command: daemon --metrics :9090
    environment:
      SQUIRREL_PASSWORD: ${SQUIRREL_PASSWORD}
    volumes:
      - ./config.yml:/root/.config/squirrel/config.yml:ro
      - squirrel-data:/backup
    ports:
      - "9090:9090"
    restart: unless-stopped
```

### Custom config path

```bash
squirrel daemon --config /etc/squirrel/config.yml --metrics :9090
```

---

## Config Reload

Send `SIGHUP` to reload the config without downtime:

```bash
kill -HUP $(pidof squirrel)
```

Profiles that are currently running are not interrupted. New schedules take effect after reload.

---

## Prometheus Metrics

When `--metrics` is configured, metrics are available at:

```
http://host:9090/metrics
```

See [Monitoring](../operations/monitoring.md) for the full list of exposed metrics.

---

## Graceful Shutdown

On `SIGTERM` or `SIGINT` (Ctrl-C):

- No new backup jobs are started.
- Running backup jobs are allowed to finish.
- The process exits cleanly.

---

## Notes

- `daemon` runs in the foreground — use your init system or process supervisor to keep it running.
- For OS-level scheduling (systemd, launchd, Task Scheduler), see [`squirrel schedule`](schedule.md).
- Each profile's `schedule:` field uses standard cron syntax or shortcuts like `@daily`.
