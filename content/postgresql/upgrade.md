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

```txt
Performing Consistency Checks on Old Live Server
------------------------------------------------
Checking cluster versions                                     ok
Checking database user is the install user                    ok
Checking database connection settings                         ok
Checking for prepared transactions                            ok
Checking for system-defined composite types in user tables    ok
Checking for reg* data types in user tables                   ok
Checking for contrib/isn with bigint-passing mismatch         ok
Checking for incompatible "aclitem" data type in user tables  ok
Checking for presence of required libraries                   fatal

Your installation references loadable libraries that are missing from the
new installation.  You can add these libraries to the new installation,
or remove the functions using them from the old installation.  A list of
problem libraries is in the file:
    /db/16/pg_upgrade_output.d/20250225T182200.708/loadable_libraries.txt
Failure, exiting
```

In this case, the upgrade is not possible because of missing libraries, you can check the `loadable_libraries.txt` file to see which libraries are missing, in this case the `postgis-3` library is missing:

```txt
could not load library "$libdir/postgis-3": ERROR:  could not access file "$libdir/postgis-3": No such file or directory
In database: test_dev1
In database: restore_test_dev1
```

After fixing the missing libraries you can run the `pg_upgrade` command again and output should be like:

```txt
Performing Consistency Checks on Old Live Server
------------------------------------------------
Checking cluster versions                                     ok
Checking database user is the install user                    ok
Checking database connection settings                         ok
Checking for prepared transactions                            ok
Checking for system-defined composite types in user tables    ok
Checking for reg* data types in user tables                   ok
Checking for contrib/isn with bigint-passing mismatch         ok
Checking for incompatible "aclitem" data type in user tables  ok
Checking for presence of required libraries                   ok
Checking database user is the install user                    ok
Checking for prepared transactions                            ok
Checking for new cluster tablespace directories               ok

*Clusters are compatible*
```

## upgrade

Run the upgrade:

```sh
/usr/lib/postgresql/16/bin/pg_upgrade \
    --old-datadir=/db/15 \
    --new-datadir=/db/16 \
    --old-bindir=/usr/lib/postgresql/15/bin \
    --new-bindir=/usr/lib/postgresql/16/bin \
    --old-options="-c config_file=/db/15/postgresql.conf" \
    --new-options="-c config_file=/db/16/postgresql.conf"\
    --link
```

> you need to stop the old postgres server before running this command `systemctl stop postgresql-15`
