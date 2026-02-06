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
  --info=progress2 --partial \
  --exclude postmaster.pid \
  --exclude postmaster.opts \
  --exclude pg_hba.conf \
  --exclude pg_ident.conf \
  --exclude postgresql.conf"

# 1. Start psql as a background coprocess so the session stays open
#    -t: tuples only (no headers)
#    -A: unaligned (no extra whitespace)
#    -q: quiet (no welcome messages)
echo "[*] Starting persistent PostgreSQL session..."
coproc PG { psql -U "$PGUSER" -d postgres -t -A -q; }

# 2. Pre-sync the data directory
echo "[*] Pre-syncing data directory (this may take a while)..."
rsync $RSYNC_OPTS "$SRC_PGDATA/" "$DST_HOST:$DST_PGDATA/"

# 3. Start backup mode and capture the label
echo "[*] Starting PostgreSQL backup mode..."
echo "SELECT pg_backup_start('rsync_migration', true);" >&"${PG[1]}"

# Read the output to ensure it started (consume the single line output of start)
read -r -u "${PG[0]}" _

# 4. Sync the data directory again to capture any changes since the first sync
echo "[*] Syncing data directory again (this should take less time)..."
rsync $RSYNC_OPTS "$SRC_PGDATA/" "$DST_HOST:$DST_PGDATA/"

# 5. Send STOP command and fetch the label file content
#    We select only the 'labelfile' column.
#    We append a sentinel string 'END_OF_LABEL' so we know when to stop reading.
echo "[*] Stopping PostgreSQL backup mode and capturing label..."
echo "SELECT labelfile FROM pg_backup_stop();" >&"${PG[1]}"
echo "SELECT 'END_OF_LABEL';" >&"${PG[1]}"

# 6. Read the label content line-by-line from the coprocess
backup_label=""
while read -r -u "${PG[0]}" line; do
    if [[ "$line" == "END_OF_LABEL" ]]; then
        break
    fi
    # Reconstruct the multiline string
    backup_label+="$line"$'\n'
done

# Close the coprocess
echo "\q" >&"${PG[1]}"

# 7. Write the label to a temp file
tmpfile=$(mktemp)
# The variable contains the exact content needed for the file
echo -n "$backup_label" > "$tmpfile"

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
