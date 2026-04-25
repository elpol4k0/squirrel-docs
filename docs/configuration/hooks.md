# Hooks

Hooks run shell commands or HTTP webhooks at defined points in the backup lifecycle.

---

## Hook Events

| Event | When it runs |
|---|---|
| `pre-backup` | Before the backup starts |
| `post-success` | After a successful backup (and retention, if configured) |
| `post-failure` | After a failed backup |

---

## Hook Actions

Each hook event accepts a list of actions. Two action types are supported:

### `command`

Execute a shell command. Arguments are passed as a list (no shell interpolation).

```yaml
hooks:
  pre-backup:
    - command: [psql, -c, "CHECKPOINT"]

  post-success:
    - command: [notify-send, "squirrel", "Backup succeeded"]
    - command: [curl, -X, POST, "https://example.com/notify"]

  post-failure:
    - command: [page-oncall, "${SQUIRREL_PROFILE}"]
```

### `webhook`

Send an HTTP POST request to a URL. The body is empty; the URL may include a status code suffix.

```yaml
hooks:
  post-success:
    - webhook: https://hc-ping.com/abc-123-uuid

  post-failure:
    - webhook: https://hc-ping.com/abc-123-uuid/fail
```

---

## Hook Environment Variables

These variables are available to `command` hooks:

| Variable | Value |
|---|---|
| `SQUIRREL_PROFILE` | Profile name being executed |
| `SQUIRREL_REPO` | Repository URL from the profile |

---

## Full Example

```yaml
profiles:
  files-home:
    repository: local
    paths:
      - /home/user/documents
    hooks:
      pre-backup:
        - command: [logger, "squirrel: starting files-home backup"]

      post-success:
        - webhook: https://monitoring.example.com/healthcheck/files-home
        - command: [logger, "squirrel: files-home backup succeeded"]

      post-failure:
        - command: [notify-send, "squirrel", "files-home backup FAILED"]
        - webhook: https://alerts.example.com/squirrel/files-home/failed

  pg-prod:
    repository: s3-prod
    type: postgres
    dsn: ${vault-prod:secret/data/postgres#replication_url}
    slot: squirrel
    hooks:
      pre-backup:
        - command: [psql, -c, "CHECKPOINT"]

      post-success:
        - webhook: https://monitoring.example.com/healthcheck/pg-prod

      post-failure:
        - webhook: https://alerts.example.com/pg-prod-backup-failed
        - command: [send-alert, --severity, critical, "--message", "pg-prod backup failed"]
```

---

## Multiple Actions per Event

Actions are executed in order. If a `command` action fails (non-zero exit code), the remaining actions for that event are still executed. Hook failures are logged but do not affect the backup exit code.

---

## Notes

- `command` actions do not use a shell — arguments are passed directly to `exec`. Use explicit `["sh", "-c", "..."]` if you need shell features.
- `webhook` actions use HTTP POST with no body and a 10-second timeout. Redirect responses are followed.
- Hook failures are recorded in logs but do not cause `squirrel run` to fail.
