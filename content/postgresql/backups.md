+++
title = "Backups"
weight = 9
+++

## Phyisical backups

Example using `pg_basebackup`:

```bash
pg_basebackup -D /tmp/pgbackup -F tar -z -X stream -c fast -P -U postgres
```

If want to run this remotely need to add the `-h` option with the IP address of the PostgreSQL server:

```bash
pg_basebackup -h <postgresql-ip> -p 5432 -D /tmp/pgbackup -F tar -z -X stream -c fast -P -U postgres
```

This will backup all the data into a tar file in `/tmp/pgbackup`. and create the file:

    - base.tar.gz
    - pg_wal.tar.gz
    - backup_manifest

To restore, you need to delete the PGDATA directory and extract the tar files:

```bash
rm -rf /var/lib/postgresql/16
```

Then extract the tar files:

```bash
tar -xzf /tmp/pgbackup/base.tar.gz -C /var/lib/postgresql/16
tar -xzf /tmp/pgbackup/pg_wal.tar.gz -C /var/lib/postgresql/16/pg_wal
```

Start the PostgreSQL server:

```bash
pg_ctl -D /var/lib/postgresql/16 start
```

## Logical backups

A logical backup captures the database contents as SQL commands that can recreate the schema and data from scratch.
It differs from a physical backup, which copies the raw data files from disk.

Logical backups are:

* Portable across PostgreSQL versions and platforms.
* Human-readable (plain SQL).
* Ideal for small to medium databases or versioned schema backups.

They are created using pg_dump (for one database) or pg_dumpall (for all databases or global objects).

### Using pg_dump

Example command to back up a single database:

```bash
pg_dump \
  -h <your IP>\
  -U <your user>\
  -F p \
  --encoding=UTF8 \
  --no-owner \
  --no-privileges \
  --blobs \
  --serializable-deferrable \
  mydatabase \
  | gzip > mydatabase_$(date +%F).sql.gz
```

| Flag                        | Purpose               | Why It Matters                       |
| --------------------------- | --------------------- | ------------------------------------ |
| `-F p`                      | Plain SQL format      | Widely portable and readable         |
| `--encoding=UTF8`           | Force UTF-8 output    | Avoids locale/charset issues         |
| `--no-owner`                | Skip ownership        | Avoids permission mismatches         |
| `--no-privileges`           | Skip GRANTs           | Simplifies cross-environment restore |
| `--blobs`                   | Include large objects | Ensures all data included            |
| `--serializable-deferrable` | Transaction-safe dump | Consistency for live systems         |
| `--no-comments`             | Skip comments         | Reduces noise, increases diffability |


To restore the backup, use the following command:

```bash
gunzip -c mydatabase_2025-10-29.sql.gz | psql -U postgres -d mydatabase
```
