+++
title = "slow queries"
weight = 4
+++

To log all slow queries add to your config `postgresql.conf`:

```config
log_min_duration_statement = 5s
```

Be careful with the option `log_duration = on` since it will log all request
(disk can be come full very quickly)

> You need to have `logging_collector = on`

Then reload postgres:

```sql
SELECT pg_reload_conf()
```

To very run:


```sql
postgres> SHOW log_min_duration_statement;
+----------------------------+
| log_min_duration_statement |
|----------------------------|
| 5s                         |
+----------------------------+
```

Test with:

```sql
postgres> select pg_sleep(5);
```

Logs will show something like:

```txt
time=2024-07-12 12:53:32 UTC, pid=2458932  db=postgres, usr=postgres, client=192.168.255.135 , app=pgcli, line=3 LOG:  duration: 5009.768 ms  statement: select pg_sleep(5)
```
