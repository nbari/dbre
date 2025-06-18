+++
title = "I/O"
weight = 1
+++

## Fio

To test speed of disks (read/write), for databases is preferred to use `fio`, is a much more accurate and better to simulate database-like I/O patterns, for example:

For MariaDB:

> Notice the block size (`bs`) is set to 16k, which is a common for MariaDB, for PostgreSQL you might want to use 8k or 4k depending on your configuration.


```bash
fio --name=mariadb-rw \
    --ioengine=libaio \
    --rw=randrw \
    --rwmixread=70 \
    --bs=16k \
    --direct=1 \
    --size=10G \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting \
    --filename=testfile
```
> Use --size=10G or adjust based on your available space — ensure it's larger than RAM to avoid cache bias.


### Random Read Only

Example simulating read-heavy OLAP load:

```bash
fio --name=mariadb-read \
    --ioengine=libaio \
    --rw=randread \
    --bs=16k \
    --direct=1 \
    --size=10G \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting \
    --filename=testfile
```

### Random Write Only

Example simulating intensive insert/update activity:

```bash
fio --name=mariadb-write \
    --ioengine=libaio \
    --rw=randwrite \
    --bs=16k \
    --direct=1 \
    --size=10G \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting \
    --filename=testfile
```


### PostgreSQL

Example for PostgreSQL with 8KB block size:

```bash

fio --name=postgres-rw \
    --ioengine=libaio \
    --rw=randrw \
    --rwmixread=70 \
    --bs=8k \
    --direct=1 \
    --size=10G \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting \
    --filename=pg_testfile
```

### WAL-like Sequential Writes (fsync)

To simulate WAL-like sequential writes, you can use:

```bash
fio --name=postgres-wal \
    --ioengine=libaio \
    --rw=write \
    --bs=8k \
    --direct=1 \
    --size=1G \
    --numjobs=1 \
    --runtime=60 \
    --fdatasync=1 \
    --group_reporting \
    --filename=pg_waltest
```

### Temporary Table I/O

To simulate temporary table I/O, (sorts, hashes, joins):

```bash
fio --name=postgres-tempio \
    --ioengine=libaio \
    --rw=randrw \
    --rwmixread=50 \
    --bs=8k \
    --direct=1 \
    --size=2G \
    --numjobs=2 \
    --runtime=60 \
    --group_reporting \
    --filename=pg_tempfile
```


## dd

You can also use `dd` for a quick test, but it is not as accurate for database workloads:

For writing:
```bash
dd if=/dev/zero of=testfile bs=1G count=1 oflag=direct status=progress
```

For reading:
```bash
dd if=testfile of=/dev/null bs=1G count=1 iflag=direct status=progress
```
