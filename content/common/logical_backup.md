+++
title = "I/O"
weight = 1
+++

## Fio

To test speed of disks (read/write), for databases is preferred to use `fio`, is a much more accurate and better to simulate database-like I/O patterns, for example:

> Notice the block size (`bs`) is set to 16k, which is a common for MariaDB, for PostgreSQL you might want to use 8k or 4k depending on your configuration.

**Note:** Make sure you have `fio` installed on your system. And also run the commands in the mountpoint of the disk you want to test (where the database files are located).

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

### Cleanup
After testing, you can clean up the test files:

```bash
rm -f pg_testfile pg_waltest pg_tempfile
```

### fio output example

```plaintext
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

postgres-rw: (g=0): rw=randrw, bs=(R) 8192B-8192B, (W) 8192B-8192B, (T) 8192B-8192B, ioengine=libaio, iodepth=1
...
fio-3.33
Starting 4 processes
postgres-rw: Laying out IO file (1 file / 10240MiB)
Jobs: 4 (f=4): [m(4)][100.0%][r=7919KiB/s,w=3443KiB/s][r=989,w=430 IOPS][eta 00m:00s]
postgres-rw: (groupid=0, jobs=4): err= 0: pid=434548: Wed Jun 18 05:36:07 2025
  read: IOPS=990, BW=7920KiB/s (8110kB/s)(464MiB/60008msec)
    slat (usec): min=2, max=329, avg= 9.89, stdev= 4.57
    clat (nsec): min=1290, max=9574.2k, avg=564570.22, stdev=307178.97
     lat (usec): min=106, max=9631, avg=574.46, stdev=307.37
    clat percentiles (usec):
     |  1.00th=[  198],  5.00th=[  223], 10.00th=[  239], 20.00th=[  273],
     | 30.00th=[  478], 40.00th=[  510], 50.00th=[  545], 60.00th=[  578],
     | 70.00th=[  635], 80.00th=[  701], 90.00th=[  898], 95.00th=[ 1090],
     | 99.00th=[ 1303], 99.50th=[ 1385], 99.90th=[ 3458], 99.95th=[ 5211],
     | 99.99th=[ 8455]
   bw (  KiB/s): min= 5008, max=10544, per=100.00%, avg=7923.79, stdev=277.47, samples=476
   iops        : min=  626, max= 1318, avg=990.47, stdev=34.68, samples=476
  write: IOPS=428, BW=3427KiB/s (3510kB/s)(201MiB/60008msec); 0 zone resets
    slat (usec): min=2, max=129, avg=11.14, stdev= 4.30
    clat (usec): min=4640, max=43184, avg=7993.70, stdev=1768.62
     lat (usec): min=4649, max=43200, avg=8004.84, stdev=1768.41
    clat percentiles (usec):
     |  1.00th=[ 4948],  5.00th=[ 5932], 10.00th=[ 6259], 20.00th=[ 6587],
     | 30.00th=[ 7046], 40.00th=[ 7373], 50.00th=[ 7767], 60.00th=[ 8160],
     | 70.00th=[ 8717], 80.00th=[ 9241], 90.00th=[10028], 95.00th=[10552],
     | 99.00th=[12256], 99.50th=[14222], 99.90th=[25822], 99.95th=[28967],
     | 99.99th=[33162]
   bw (  KiB/s): min= 2864, max= 3760, per=100.00%, avg=3428.71, stdev=36.74, samples=476
   iops        : min=  358, max=  470, avg=428.59, stdev= 4.59, samples=476
  lat (usec)   : 2=0.01%, 100=0.01%, 250=9.69%, 500=15.89%, 750=33.05%
  lat (usec)   : 1000=6.38%
  lat (msec)   : 2=4.67%, 4=0.07%, 10=27.18%, 20=2.99%, 50=0.07%
  cpu          : usr=0.15%, sys=0.53%, ctx=85675, majf=0, minf=45
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=59408,25708,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=7920KiB/s (8110kB/s), 7920KiB/s-7920KiB/s (8110kB/s-8110kB/s), io=464MiB (487MB), run=60008-60008msec
  WRITE: bw=3427KiB/s (3510kB/s), 3427KiB/s-3427KiB/s (3510kB/s-3510kB/s), io=201MiB (211MB), run=60008-60008msec

Disk stats (read/write):
  vda: ios=59908/26311, merge=0/18, ticks=33860/218043, in_queue=254055, util=97.19%
```

From the output, you can see:
- **IOPS**: Input/Output Operations Per Second, indicating how many read/write operations are performed per second. IOPS: ~990 read, ~428 write = ~1,418 total IOPS
- **BW**: Bandwidth, showing the amount of data read/written per second. BW: ~7920KiB/s read, ~3427KiB/s write = ~11,347KiB/s total
- **Latency**: The time taken for an I/O operation to complete, with percentiles showing distribution. Latency: avg read ~574.46us, avg write ~8004.84us (reads are faster than writes, which is typical for many workloads)
- **CPU Usage**: Indicates how much CPU time was spent on I/O operations. CPU: usr=0.15%, sys=0.53% (low CPU usage, indicating efficient I/O operations)
- **IO Depth**: The number of I/O operations that can be in progress at once. IO depths: 1=100.0% (indicating single-threaded I/O, which is common for many database workloads, In PostgreSQL, each backend process (i.e., per connection) typically performs synchronous, blocking I/O — issuing one request at a time, waiting for completion, then issuing the next. So IO depths: 1=100.0% in fio does match the behavior of a single PostgreSQL connection, not because of max_connections = 1, but because each connection handles I/O in a single-threaded, synchronous fashion.


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
