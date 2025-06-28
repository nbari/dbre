+++
title = "Lock Wait Timeout"
weight = 3
+++

In cases where you get an error like this `Lock wait timeout exceeded; try
restarting transaction`, it indicates that a transaction is waiting for a lock
to be released, but the wait time has exceeded the configured limit.

To resolve this issue, you can try the following steps:
**Increase the `innodb_lock_wait_timeout`**: This setting controls how long a transaction will wait for a lock before timing out. You can increase this value in your MariaDB configuration file (usually `my.cnf`) under the `[mysqld]` section:

```ini
[mysqld]
innodb_lock_wait_timeout = 120  # Set to 120 seconds, adjust as needed
```
