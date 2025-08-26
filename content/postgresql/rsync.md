+++
title = "rsync"
weight = 10
+++

# Move PostgreSQL data directory with `rsync`

You can use rsync if need to move your PostgreSQL data directory to another
location. This is useful if you are running out of space on the current disk or
if you want to move the data directory to a faster disk.

This can be done without stopping the PostgreSQL server, the steps are as
follows:

1. `rsync` the data directory to the new location.
2. Once almost all the data is copied, you can either stop the PostgreSQL server (advised) and sync the ramaining data or take a `pg_backup_start`,  rsync the remaining data and then take a `pg_backup_stop` and then sync again.
3. update the `data_directory` parameter in `postgresql.conf` of the new location
4. start the PostgreSQL server

Here is an example of how to do this, from the active server to a new server:

```bash
rsync -aHAXx --numeric-ids --delete \
  --exclude postmaster.pid \
  --exclude postmaster.opts \
  /var/lib/postgresql/16 \
  newserver:/var/lib/postgresql/16
 ```
> You may also exclude the pg_ha.conf, pg_indent.conf and the postgresql.conf if you want to keep the old configuration files.

Once the rsync is almost done, create the backup label and sync again:

```sql
SELECT pg_backup_start('my_backup', true);
```
> The `true` will try to create the checkpoint faster

This is useful because PostgreSQL does not flush everything instantly to disk; many pages live in shared buffers and are written to disk later. By taking a backup label, you ensure that the data is in a consistent state.

`pg_backup_start` performs a checkpoint, starts a full-page write in WAL, and creates a backup label file in the data directory. This ensures that the data copied is consistent and can be used for recovery if needed.

Then run the rsync again to copy the remaining data and when finished run:

```sql
SELECT pg_backup_stop();
```

This will remove the backup label file and stop the full-page writes in WAL.

run the `rsync` again to copy the remaining data.


Finally, update the `data_directory` parameter in `postgresql.conf` on the new server.

# script

Here is a script that automates the process:

```bash
#!/bin/bash
set -euo pipefail

SRC_PGDATA="/var/lib/postgresql/15/main"
DST_HOST="newserver"
DST_PGDATA="/var/lib/postgresql/15/main"
PGUSER="postgres"

RSYNC_OPTS="-aHAXx --numeric-ids --delete \
  --exclude postmaster.pid \
  --exclude postmaster.opts \
  --exclude pg_hba.conf \
  --exclude pg_ident.conf \
  --exclude postgresql.conf"

echo "[*] Pre-syncing data directory (this may take a while)..."
rsync $RSYNC_OPTS "$SRC_PGDATA/" "$DST_HOST:$DST_PGDATA/"

echo "[*] Starting PostgreSQL backup mode..."
psql -U $PGUSER -d postgres -c "SELECT pg_backup_start('rsync_migration', true);"

echo "[*] Syncing data directory again (this should take less time)..."
rsync $RSYNC_OPTS "$SRC_PGDATA/" "$DST_HOST:$DST_PGDATA/"

echo "[*] Stopping PostgreSQL backup mode..."
psql -U $PGUSER -d postgres -c "SELECT pg_backup_stop();"

echo "[*] Last sync of data directory..."
rsync $RSYNC_OPTS "$SRC_PGDATA/" "$DST_HOST:$DST_PGDATA/"

echo "[*] Fixing ownership on destination..."
ssh "$DST_HOST" "sudo chown -R postgres:postgres $DST_PGDATA"

echo
echo "[*] Migration complete!"
echo "Now you can start PostgreSQL on $DST_HOST with:"
echo "    pg_ctl -D $DST_PGDATA start"
echo
echo "Don't forget to update postgresql.conf if needed."
```
