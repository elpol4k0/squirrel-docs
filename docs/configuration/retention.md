# Retention Policies

Retention policies control how many snapshots are kept. squirrel implements a bucket-based strategy that matches the approach used by restic and Borg.

---

## Retention Fields

```yaml
retention:
  keep-last: 5        # Keep N most recent snapshots
  keep-hourly: 24     # Keep 1 per hour for N hours
  keep-daily: 7       # Keep 1 per day for N days
  keep-weekly: 4      # Keep 1 per week for N weeks
  keep-monthly: 12    # Keep 1 per month for N months
  keep-yearly: 3      # Keep 1 per year for N years
  prune: true         # Delete unreferenced blobs after applying policy
```

All fields are optional. At least one `keep-*` must be set to use a retention policy.

---

## How the Policy Works

Snapshots are evaluated from **newest to oldest**. For each bucket type, the **first** snapshot in each time period (calendar hour, day, week, month, year) is kept. All others in that period are dropped.

`keep-last` always keeps the N newest snapshots regardless of which buckets they fill.

### Example

With `keep-last: 3`, `keep-daily: 7`, `keep-weekly: 4`:

- The 3 newest snapshots are always kept.
- For each of the last 7 calendar days, the newest snapshot in that day is kept.
- For each of the last 4 calendar weeks (Mon–Sun), the newest snapshot in that week is kept.
- Any snapshot not selected by any rule is removed.

---

## `prune` Flag

```yaml
retention:
  keep-last: 7
  prune: true
```

When `prune: true` is set, squirrel automatically runs [`squirrel prune`](../commands/prune.md) after applying the retention policy. This frees storage by removing blob data that no longer belongs to any snapshot.

Without `prune: true`, only snapshot manifests are removed; blob data remains until `squirrel prune` is run manually.

---

## Global Defaults

Set default retention applied to all profiles in `defaults:`:

```yaml
defaults:
  retention:
    keep-last: 3
    keep-daily: 7
    keep-weekly: 4
    keep-monthly: 12
    keep-yearly: 3
    prune: true
```

Individual profiles override these defaults field-by-field:

```yaml
profiles:
  files-home:
    retention:
      keep-daily: 30    # overrides defaults.retention.keep-daily
      # keep-weekly, keep-monthly, keep-yearly from defaults still apply
```

---

## Applying Retention Manually

```bash
# Preview what would be removed
squirrel forget --repo /backup/myrepo \
  --keep-last 5 --keep-daily 30 --dry-run

# Apply and prune
squirrel forget --repo /backup/myrepo \
  --keep-last 5 --keep-daily 30 --prune
```

---

## Common Policies

### Minimal (development / staging)

```yaml
retention:
  keep-last: 3
  keep-daily: 7
  prune: true
```

### Standard (production)

```yaml
retention:
  keep-last: 5
  keep-daily: 14
  keep-weekly: 8
  keep-monthly: 24
  keep-yearly: 5
  prune: true
```

### Long-term compliance

```yaml
retention:
  keep-daily: 30
  keep-monthly: 36
  keep-yearly: 7
  prune: true
```
