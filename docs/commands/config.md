# squirrel config

Manage the squirrel configuration file.

---

## Default Config File Location

| Platform | Path |
|---|---|
| Linux | `~/.config/squirrel/config.yml` |
| macOS | `~/.config/squirrel/config.yml` |
| Windows | `%APPDATA%\squirrel\config.yml` |

All subcommands accept `--config <path>` to override the default location.

---

## Subcommands

### squirrel config init

Create an initial config file from a template.

```
squirrel config init [--config <path>] [--force]
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--config` | string | platform default | Output path for the config file. |
| `--force` | bool | false | Overwrite existing file. |

Creates a commented template with example sections for repositories, secrets, and profiles.

---

### squirrel config validate

Parse and validate a config file, reporting any errors.

```
squirrel config validate [--config <path>]
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--config` | string | platform default | Config file to validate. |

Output:

```
Config valid.
```

or:

```
ERROR: profiles.pg-prod: repository "s3-prod" not defined
```

---

### squirrel config show

Show the fully resolved configuration for a profile, including expanded secret values.

```
squirrel config show <profile> [--config <path>] [--reveal]
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--config` | string | platform default | Config file to read. |
| `--reveal` | bool | false | Show secret values in plain text. Without this flag, secrets are masked as `***`. |

| Argument | Description |
|---|---|
| `<profile>` | Profile name to display. |

---

## Examples

### Create a starter config

```bash
squirrel config init
```

### Validate after editing

```bash
squirrel config validate
```

### Inspect a profile

```bash
squirrel config show pg-prod
```

Sample output (secrets masked):

```yaml
repository: s3-prod
type: postgres
dsn: ***
slot: squirrel
tags: [postgres, production]
schedule: "0 3 * * *"
retention:
  keep-last: 5
  keep-daily: 14
  prune: true
```

### Reveal secrets for debugging

```bash
squirrel config show pg-prod --reveal
```

!!! warning
    `--reveal` prints secrets in plain text to stdout. Use with caution in shared environments or CI logs.

### Use custom config path

```bash
squirrel config validate --config /etc/squirrel/config.yml
squirrel config show pg-prod --config /etc/squirrel/config.yml
```
