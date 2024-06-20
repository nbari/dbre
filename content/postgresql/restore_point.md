+++
title = "pg_create_restore_point"
weight = 3
+++

In PostgreSQL, ensuring data integrity and ease of recovery during maintenance
or deployment activities is crucial. One effective way to safeguard your
database is by using the `pg_create_restore_point` function. This function
creates a named, durable restore point which can be used to recover the
database to a specific state using Point-in-Time Recovery (PITR). Here's a
brief introduction on how to utilize pg_create_restore_point before performing
maintenance or deploying changes.

## Create

Create a restore point named `RP1`:

```sql
SELECT pg_create_restore_point('RP1');
```

After doing this, you can restore that that point `RP1`.

## Stop

For restoring, first stop postgres:

```sh
systemctl stop postgresql.service
```

## Restore

```sh
pgbackrest restore --stanza=stanza_name --type=name --target=RP1
```

## recovery_target_name

Check the `$PGDATA/postgres.auto.conf`

```conf
restore_command = 'pgbackrest --stanza=stanza_name archive-get %f "%p"'
recovery_target_name = 'RP1'
```

## Start

Start postgresql and check the data:

```sh
systemctl stop postgresql.service
```
