+++
title = "Latency"
+++

To test connectivity, and latency you can use `pgbench` for example:

Create a file named `query.sql` with contents:

```sql
SELECT 1
```

Then run `pgbench` with:

    pgbench -h <IP> -U <user> -d <database> -c 1 -T 10 -f query.sql

* -c is for the number of clients
* -T to run it for 10 seconds

The output will be something like:

```text
transaction type: query.sql
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 4746
number of failed transactions: 0 (0.000%)
latency average = 2.102 ms
initial connection time = 28.091 ms
tps = 475.738156 (without initial connection time)
```
