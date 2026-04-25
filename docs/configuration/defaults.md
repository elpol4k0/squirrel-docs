# Global Defaults

The `defaults:` section sets values that apply to all profiles unless overridden.

---

## Structure

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

---

## Supported Default Fields

| Field | Description |
|---|---|
| `retention` | Default retention policy applied to all profiles. See [Retention](retention.md) for all sub-fields. |

---

## Override Behavior

Profile-level `retention` fields override defaults field-by-field. Unspecified fields in a profile fall back to the default.

### Example

```yaml
defaults:
  retention:
    keep-daily: 7
    keep-weekly: 4
    keep-monthly: 12
    prune: true

profiles:
  files-home:
    retention:
      keep-daily: 30    # overrides default keep-daily
      # keep-weekly: 4 from defaults
      # keep-monthly: 12 from defaults
      # prune: true from defaults

  pg-staging:
    retention:
      keep-last: 3
      keep-daily: 7
      prune: true       # explicitly set; defaults are ignored for this profile
```

---

## Notes

- Currently, only `retention` is supported under `defaults:`.
- If no `defaults:` section is present and no retention is set in a profile, no retention policy is applied — old snapshots accumulate indefinitely.
- It is good practice to always set a default retention policy to prevent unbounded storage growth.
