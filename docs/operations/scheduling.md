# Scheduling

squirrel provides two approaches to automated scheduling: a **built-in daemon scheduler** and **OS-native scheduler integration**.

``` mermaid
flowchart LR
    subgraph Daemon["squirrel daemon (containers / VMs)"]
        CRON[built-in cron\nscheduler] --> RUN[squirrel run\nprofile]
        RUN --> METRICS[Prometheus\nmetrics :9090]
    end
    subgraph OS["OS-native scheduler"]
        SYSTEMD[systemd timer] & LAUNCHD[launchd plist] & TASKS[Task Scheduler] --> RUN2[squirrel run\nprofile]
    end
    CONFIG[config.yml\nschedule: cron] --> Daemon & OS
```

---

## Option 1: squirrel daemon (recommended for containers)

`squirrel daemon` runs as a foreground process with a built-in cron scheduler. It reads `schedule:` fields from all profiles and fires them at the configured times.

```bash
squirrel daemon --metrics :9090
```

### Advantages

- Single process, no external scheduler required.
- Works inside Docker/Kubernetes without systemd.
- Live config reload via `SIGHUP`.
- Built-in Prometheus metrics.

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
    restart: unless-stopped
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: squirrel
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: squirrel
          image: ghcr.io/elpol4k0/squirrel:latest
          args: ["daemon", "--metrics", ":9090"]
          env:
            - name: SQUIRREL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: squirrel-secret
                  key: password
          volumeMounts:
            - name: config
              mountPath: /root/.config/squirrel
```

---

## Option 2: OS-Native Scheduler

Install OS-level tasks for individual profiles using `squirrel schedule install`.

### Linux (systemd)

```bash
# Install (reads schedule: from profile config)
squirrel schedule install pg-daily

# Enable and start the timer
systemctl --user enable --now squirrel-pg-daily.timer

# Check status
systemctl --user status squirrel-pg-daily.timer

# View logs
journalctl --user -u squirrel-pg-daily.service
```

Generated files:
```
~/.config/systemd/user/squirrel-pg-daily.service
~/.config/systemd/user/squirrel-pg-daily.timer
```

For system-wide installation (run as root):
```bash
sudo squirrel schedule install pg-daily --config /etc/squirrel/config.yml
sudo systemctl enable --now squirrel-pg-daily.timer
```

### macOS (launchd)

```bash
squirrel schedule install pg-daily

# Load the plist
launchctl load ~/Library/LaunchAgents/io.squirrel.pg-daily.plist

# Check status
launchctl print gui/$(id -u)/io.squirrel.pg-daily
```

Generated file:
```
~/Library/LaunchAgents/io.squirrel.pg-daily.plist
```

### Windows (Task Scheduler)

```powershell
squirrel schedule install pg-daily

# Verify in Task Scheduler
Get-ScheduledTask -TaskName "squirrel-pg-daily"

# Run immediately for testing
Start-ScheduledTask -TaskName "squirrel-pg-daily"
```

Created entry: `Task Scheduler → \squirrel-pg-daily`

---

## Cron Expression Reference

squirrel uses standard 5-field cron syntax:

```
┌───── minute    (0–59)
│ ┌──── hour      (0–23)
│ │ ┌─── day       (1–31)
│ │ │ ┌── month     (1–12)
│ │ │ │ ┌─ weekday   (0–7, 0 and 7 = Sunday)
│ │ │ │ │
* * * * *
```

### Common Schedules

| Expression | When |
|---|---|
| `0 2 * * *` | Daily at 02:00 |
| `30 1 * * *` | Daily at 01:30 |
| `0 3 * * 0` | Weekly on Sunday at 03:00 |
| `0 */6 * * *` | Every 6 hours |
| `0 0 1 * *` | Monthly on the 1st at midnight |
| `*/15 * * * *` | Every 15 minutes |
| `@daily` | Daily at midnight |
| `@weekly` | Weekly on Sunday at midnight |
| `@monthly` | Monthly on the 1st at midnight |
| `@hourly` | Every hour |

### In Config Profile

```yaml
profiles:
  pg-prod:
    repository: s3-prod
    type: postgres
    dsn: ${env:PG_DSN}
    slot: squirrel
    schedule: "0 3 * * *"     # Daily at 03:00

  pg-prod-offsite:
    extends: pg-prod
    repository: sftp-offsite
    schedule: "0 5 * * 0"     # Weekly on Sunday at 05:00

  files-home:
    repository: local
    paths: [/home/user]
    schedule: "@daily"
```

---

## Managing Schedules

```bash
# List installed schedules
squirrel schedule list

# Remove a schedule
squirrel schedule remove pg-daily

# Remove all squirrel schedules
squirrel schedule list | xargs squirrel schedule remove
```
