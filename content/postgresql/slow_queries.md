+++
title = "log_min_duration_statement"
weight = 4
+++

To log all slow queries add to your config:

```config
log_min_duration_statement = 5s
```

Be careful with the option `log_duration = on` since it will log all request
(disk can be come full very quickly)

> You need to have `logging_collector = on`

+----------------------------+
| log_min_duration_statement |
|----------------------------|
| 5s                         |
+----------------------------+

Then postgres:

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


