+++
title = "pg_create_restore_point"
weight = 3
+++

## Backup using a Restore point

An easy way to create a restore point is by using:

```sql
SELECT pg_create_restore_point('RP1');
```

After doing this, you can restore that that point `RP1`, first stop postgres:

```sh
systemctl stop postgresql.service
```

## Restore

```sh
pgbackrest restore --stanza=<stanza_name> --type=name --target=RP1
```

## recovery_target_name

Check the `$PGDATA/postgres.auto.conf`

```sh
restore_command = 'pgbackrest --stanza=<stanza_name> archive-get %f "%p"'
recovery_target_name = 'RP1'
```

## Start postgres

Start posgres and check the data:

```sh
systemctl stop postgresql.service
```
