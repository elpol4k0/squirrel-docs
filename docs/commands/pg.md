# squirrel pg

PostgreSQL management commands.

---

## squirrel pg drop-slot

Drop a PostgreSQL physical replication slot.

```
squirrel pg drop-slot --dsn <dsn> --slot <name>
```

### Flags

| Flag | Type | Required | Description |
|---|---|---|---|
| `--dsn` | string | **Yes** | PostgreSQL DSN with replication access. |
| `--slot` | string | **Yes** | Name of the replication slot to drop. |

---

## Why Drop a Slot?

PostgreSQL replication slots retain WAL on disk until the slot is consumed. If squirrel stops streaming WAL for an extended period (e.g. backup job paused), the primary accumulates WAL indefinitely, which can fill the disk.

Drop unused slots to allow WAL cleanup:

```bash
squirrel pg drop-slot \
  --dsn "postgres://squirrel:secret@localhost/postgres?sslmode=disable" \
  --slot squirrel
```

---

## Listing Existing Slots

Use `psql` to list active replication slots:

```bash
psql -U squirrel -c "SELECT slot_name, active, restart_lsn FROM pg_replication_slots;"
```

---

## Notes

- Dropping a slot while WAL streaming is active terminates the streaming connection.
- A new slot is created automatically the next time `squirrel backup postgres` runs.
- If the slot does not exist, the command returns an error.
