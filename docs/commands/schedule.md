# squirrel schedule

Install, remove, and list OS-level scheduled tasks for squirrel profiles.

squirrel integrates with the native scheduler on each platform:

| Platform | Scheduler |
|---|---|
| Linux | systemd timers + service units |
| macOS | launchd plists |
| Windows | Task Scheduler |

---

## Subcommands

### squirrel schedule install

Install an OS-level scheduled task for one or more profiles.

```
squirrel schedule install <profile> [profile...] [flags]
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--config` | string | platform default | Path to config file. |

The cron expression is taken from the `schedule:` field in the profile definition.

#### Generated Files

=== "Linux (user)"
    ```
    ~/.config/systemd/user/squirrel-<profile>.service
    ~/.config/systemd/user/squirrel-<profile>.timer
    ```

=== "Linux (root)"
    ```
    /etc/systemd/system/squirrel-<profile>.service
    /etc/systemd/system/squirrel-<profile>.timer
    ```

=== "macOS"
    ```
    ~/Library/LaunchAgents/io.squirrel.<profile>.plist
    ```

=== "Windows"
    ```
    Task Scheduler → \squirrel-<profile>
    ```

---

### squirrel schedule remove

Remove the OS-level scheduled task for one or more profiles.

```
squirrel schedule remove <profile> [profile...] [flags]
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--config` | string | platform default | Path to config file. |

---

### squirrel schedule list

List all installed squirrel scheduled tasks.

```
squirrel schedule list
```

---

## Examples

### Install a schedule for pg-daily

```bash
# Profile must have a schedule: field
squirrel schedule install pg-daily
```

=== "Linux (enable timer)"
    ```bash
    systemctl --user enable --now squirrel-pg-daily.timer
    systemctl --user status squirrel-pg-daily.timer
    ```

=== "macOS"
    ```bash
    launchctl load ~/Library/LaunchAgents/io.squirrel.pg-daily.plist
    ```

### List installed schedules

```bash
squirrel schedule list
```

### Remove a schedule

```bash
squirrel schedule remove pg-daily
```

---

## Cron Expression Format

squirrel uses standard 5-field cron syntax:

```
┌───── minute (0–59)
│ ┌──── hour (0–23)
│ │ ┌─── day of month (1–31)
│ │ │ ┌── month (1–12)
│ │ │ │ ┌─ day of week (0–7, 0 or 7 = Sunday)
│ │ │ │ │
* * * * *
```

Shortcuts:

| Shortcut | Equivalent |
|---|---|
| `@hourly` | `0 * * * *` |
| `@daily` | `0 0 * * *` |
| `@weekly` | `0 0 * * 0` |
| `@monthly` | `0 0 1 * *` |

Examples:

```yaml
schedule: "0 2 * * *"       # Daily at 02:00
schedule: "30 3 * * 0"      # Weekly on Sunday at 03:30
schedule: "0 */6 * * *"     # Every 6 hours
schedule: "@daily"           # Daily at midnight
```

---

## Notes

- The `schedule:` field is required in the profile for `schedule install` to work.
- As an alternative to OS-level scheduling, use [`squirrel daemon`](daemon.md), which has a built-in scheduler suitable for containers.
