+++
title = "Backups"
weight = 9
+++

## Phyisical backups

Example using `pg_basebackup`:

```bash
pg_basebackup -D /tmp/pgbackup -F tar -z -X stream -c fast -P -U postgres
```

This will backup all the data in the PostgreSQL cluster to a tar file in `/tmp/pgbackup`. and create the file:

    * base.tar.gz
    * pg_wal.tar.gz
    * backup_manifest

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
