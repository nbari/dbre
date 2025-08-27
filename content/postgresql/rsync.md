+++
title = "rsync"
weight = 10
+++

## Rsync

You can use `rsync` if need to move your PostgreSQL data directory to another
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
rsync -aHAXx --numeric-ids --delete --info=progress2 --inplace --partial \
  -e "ssh -T -c aes128-gcm@openssh.com -o Compression=no -x" \
  --exclude postmaster.pid \
  --exclude postmaster.opts \
  /opt/postgres/16 \
  newserver:/opt/postgres/
 ```
> You may also exclude the pg_ha.conf, pg_indent.conf and the postgresql.conf if you want to keep the old configuration files.

The ssh options explained (This often doubles throughput on LANs):

    -T disables SSH pseudo-tty (tiny win).
    -c aes128-gcm@openssh.com uses a very fast cipher (assuming OpenSSH ≥ 6.2).
    -o Compression=no avoids double compression overhead.
    -x disables X11 forwarding (tiny win).


Once the rsync is almost done, create the backup label and sync again:

```sql
SELECT pg_backup_start('rsync_migration', true);
```
> The `true` will try to create the checkpoint faster

This is useful because PostgreSQL does not flush everything instantly to disk; many pages live in shared buffers and are written to disk later. By taking a backup label, you ensure that the data is in a consistent state.

`pg_backup_start` performs a checkpoint, starts a full-page write in WAL, and creates a backup label file in the data directory. This ensures that the data copied is consistent and can be used for recovery if needed.

Then run the rsync again to copy the remaining data and when finished run:

```sql
SELECT pg_backup_stop();
```

Save the output of `pg_backup_stop()`, as it contains the backup_label and the
WAL segment name needed to make the backup consistent. This tells you which WAL
files must be copied to the destination for recovery. The output looks like:

```txt
postgres=# SELECT pg_backup_stop();
NOTICE:  all required WAL segments have been archived
                                  pg_backup_stop
-----------------------------------------------------------------------------------
 (20E0/7B000138,"START WAL LOCATION: 20E0/7B000028 (file 00000008000020E00000007B)+
 CHECKPOINT LOCATION: 20E0/7B000060                                               +
 BACKUP METHOD: streamed                                                          +
 BACKUP FROM: primary                                                             +
 START TIME: 2025-08-27 13:51:42 UTC                                              +
 LABEL: rsync_migration                                                                 +
 START TIMELINE: 8                                                                +
 ","")
(1 row)

```


In the new server create the `backup_label` file in the new data directory `$PGDATA` with the content from the output of `pg_backup_stop()`., from the example above it would be:

```txt
START WAL LOCATION: 20E0/7B000028 (file 00000008000020E00000007B)
CHECKPOINT LOCATION: 20E0/7B000060
BACKUP METHOD: streamed
BACKUP FROM: primary
START TIME: 2025-08-27 13:51:42 UTC
LABEL: rsync_migration
START TIMELINE: 8
```

Opionaly, run the `rsync` again to copy the remaining data.


Finally, update the `data_directory` parameter in `postgresql.conf` on the new server, ensure the `backup_label` file is present in the new data directory, and start PostgreSQL:


## Script

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

echo "[*] Stopping PostgreSQL backup mode and capturing label..."
backup_info=$(psql -U $PGUSER -d postgres -t -A -c "SELECT pg_backup_stop();")

# Extract the formatted part from the output
backup_label=$(echo "$backup_info" | sed 's/^[ (]\|[",)]$//g')

# Write the label into a temp file
tmpfile=$(mktemp)
echo "$backup_label" | tr '+' '\n' > "$tmpfile"

echo "[*] Last sync of data directory..."
rsync $RSYNC_OPTS "$SRC_PGDATA/" "$DST_HOST:$DST_PGDATA/"

echo "[*] Installing backup_label on destination..."
scp "$tmpfile" "$DST_HOST:$DST_PGDATA/backup_label"
rm -f "$tmpfile"

echo "[*] Fixing ownership on destination..."
ssh "$DST_HOST" "sudo chown -R postgres:postgres $DST_PGDATA"

echo
echo "[*] Migration complete!"
echo "Now you can start PostgreSQL on $DST_HOST with:"
echo "    pg_ctl -D $DST_PGDATA start"
echo
echo "Don't forget to update postgresql.conf if needed."
```
