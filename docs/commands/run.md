# squirrel run

Execute one or more backup profiles defined in the config file.

```
squirrel run <profile> [profile...] [flags]
```

---

## Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--config` | string | platform default | Path to config file. |
| `--parallel` | int | 1 | Number of profiles to run concurrently. |

---

## Arguments

| Argument | Description |
|---|---|
| `<profile>` | One or more profile names to execute. Must not be abstract profiles (names starting with `_` by convention). |

---

## Profile Execution

For each profile, `run` performs:

1. Resolves secrets from all configured providers.
2. Opens the repository specified by `repository:`.
3. Executes the appropriate backup command based on `type:` (`files`, `postgres`, `mysql`).
4. Applies the retention policy from `retention:` (if configured).
5. Prunes unreferenced blobs if `retention.prune: true`.
6. Runs hooks (`pre-backup`, `post-success`, `post-failure`).

---

## Examples

### Run a single profile

```bash
squirrel run pg-daily
```

### Run multiple profiles sequentially

```bash
squirrel run pg-daily files-home mysql-weekly
```

### Run multiple profiles in parallel

```bash
squirrel run pg-daily files-home --parallel 2
```

### Use a custom config file

```bash
squirrel run pg-daily --config /etc/squirrel/config.yml
```

---

## Exit Codes

| Code | Meaning |
|---|---|
| 0 | All profiles completed successfully. |
| 1 | One or more profiles failed. |

When `--parallel` is greater than 1, all profiles run before a non-zero exit code is returned.

---

## Environment Variables Available in Hooks

| Variable | Value |
|---|---|
| `SQUIRREL_PROFILE` | Name of the profile being executed |
| `SQUIRREL_REPO` | Repository URL from the profile |

---

## Notes

- Abstract profiles (with `abstract: true`) cannot be run directly with `squirrel run`.
- Profile inheritance (`extends:`) is resolved before execution — the child profile receives all parent settings, with its own fields taking precedence.
- Use [`squirrel daemon`](daemon.md) to run profiles on a schedule automatically.
