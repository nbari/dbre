+++
title = "checkpoint"
weight = 11
+++

## Checkpoint


A checkpoint is a critical operation in PostgreSQL that ensures data integrity
and durability. It involves writing all modified data pages (dirty pages) from
the shared buffer cache to the disk, as well as updating the write-ahead log
(WAL) to reflect these changes. Checkpoints help minimize recovery time in case
of a crash by reducing the amount of WAL that needs to be replayed.



### Relationship between `wal_segment_size` and `max_wal_size`

`wal_segment_size` → the size of a single WAL file on disk.

`max_wal_size` → the target maximum total WAL space before a checkpoint triggers.

For example if `wal_segment_size` is 16MB and `max_wal_size` is 1GB, then a
checkpoint will be triggered after approximately 64 WAL files (1GB / 16MB) have
been generated.

if `wal_segment_size` is increased, for example to 64MB, and `max_wal_size`
remains at 1GB, then a checkpoint will be triggered after approximately 16 WAL
files (1GB / 64MB) have been generated.

This means that with a larger `wal_segment_size`, checkpoints may occur less
frequently in terms of the number of WAL files, but the total amount of WAL
data written before a checkpoint remains the same as defined by `max_wal_size`.

### Where the WAL files are stored

WAL files are stored in the `pg_wal` directory within the PostgreSQL data:

```sql
SHOW data_directory;
```

list the files in the `pg_wal` directory:

```bash
ls -lh $PGDATA/pg_wal
```

Even with a long checkpoint_timeout, PostgreSQL may still trigger a checkpoint if WAL reaches `max_wal_size`, example:

```
wal_segment_size = 64MB
max_wal_size = 256MB
checkpoint_timeout = 30min
```

You'll see `~4 WAL` files fill up before a checkpoint is forced (4 × 64 MB = 256 MB).


### Test Checkpoint settings

```sql
ALTER SYSTEM SET checkpoint_timeout = '30s';
ALTER SYSTEM SET max_wal_size = '256MB';
ALTER SYSTEM SET log_checkpoints = on;
```

Then reload PostgreSQL:

```sql
SELECT pg_reload_conf();
```

Use pgbench to generate some load, first initialize the database:

```bash
pgbench -i -s 50 postgres
```

Then run pgbench with 10 clients for 5 minutes:

```bash
pgbench -c 10 -T 300 postgres
```

### Monitor current WAL usage:

You can check the current WAL usage and the last checkpoint location using the following SQL query:

```sql
-- MB since last checkpoint
SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), checkpoint_lsn)/1024/1024 AS mb_wal_since_checkpoint
FROM pg_control_checkpoint(); \watch 1
```

And also list the `pg_wal` directory to see the number of WAL files:

```bash
watch ls -lh $PGDATA/pg_wal
```

You can adjust the `checkpoint_timeout` and `max_wal_size` settings to see how
they affect the frequency of checkpoints and the number of WAL files generated,
also theck the PostgreSQL logs for checkpoint messages if `log_checkpoints` is
enabled.
