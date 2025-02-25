+++
title = "Upgrade major version"
weight = 8
+++

Before upgrading take a backup of your data

## backup

Create full backup:

```sh
pgbackrest --stanza=standalone --type=full backup
```

Install the postgres version you want to upgrade to, in this case, I am
upgrading from `15` to `16`, the current `PGDATA` is `/db/15` and the new `PGDATA` will
be `/db/16`.

```sh
apt install postgresql-16
```

## initdb

Initialize the new postgres data directory:

```sh
/usr/lib/postgresql/16/bin/pg_ctl initdb -D /db/16
```

## pg_hba.conf

Update in both `/db/15/pg_hba.conf` and `/db/16/pg_hba.conf` to use `trust` instead of  `scram-sha-256` and comment al the `host` entries to temporally restrict remote connections:

```conf
local   all    all    trust
# host    all    all    0.0.0.0/0    scram-sha-256
```

## upgrade check

Check if the upgrade is possible:

```sh
/usr/lib/postgresql/16/bin/pg_upgrade \
    --old-datadir=/db/15 \
    --new-datadir=/db/16 \
    --old-bindir=/usr/lib/postgresql/15/bin \
    --new-bindir=/usr/lib/postgresql/16/bin \
    --link --check
```

> You need to run this command as the `postgres` user

The output can be something like:
