+++
title = "Upgrade major version"
weight = 8
+++

Before upgrading take a backup of your data

## backup

Create full backup:

```sh
pgbackrest --stanza=<stanza_name> --type=full backup
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

In this case, the upgrade is not possible because of missing libraries, you can
check the `loadable_libraries.txt` file to see which libraries are missing, in
this case the `postgis-3` library is missing:

```txt
could not load library "$libdir/postgis-3": ERROR:  could not access file "$libdir/postgis-3": No such file or directory
In database: test_dev1
In database: restore_test_dev1
```

It can happen that you end in a loop because when installing the new libraries
it will remove the old extensions in the `../15/lib`, for this you can copy the
`/usr/lib/postgresql/15/lib` to `/usr/lib/postgresql/15/lib.old`:

```sh
cp -r /usr/lib/postgresql/15/lib /usr/lib/postgresql/15/lib.old
```

Do now the upgrade of the libraries for the new version, in this case, the
`postgis-3` library and when finished move back the `lib.old`  to `lib`:

```sh
mv /usr/lib/postgresql/15/lib.old /usr/lib/postgresql/15/lib
```

This is because depending on the OS the libraries will remove the old extensions
when installing the new ones, this can cause the `pg_upgrade` to fail.

After you have installed the new libraries you can run the `pg_upgrade` command again:

> in RH based systems the libraries are in `/usr/pgsql-15/lib` and `/usr/pgsql-16/lib` respectively, and may need to run: `dnf upgrade --allowerasing *.rpm` and then `dnf install *.rpm` in the directory containing the new libraries.


After fixing the missing libraries you can run the `pg_upgrade` command again
and output should be like:

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

The output should be like:

```txt
Performing Consistency Checks
-----------------------------
Checking cluster versions                                     ok
Checking database user is the install user                    ok
Checking database connection settings                         ok
Checking for prepared transactions                            ok
Checking for system-defined composite types in user tables    ok
Checking for reg* data types in user tables                   ok
Checking for contrib/isn with bigint-passing mismatch         ok
Checking for incompatible "aclitem" data type in user tables  ok
Creating dump of global objects                               ok
Creating dump of database schemas
                                                              ok
Checking for presence of required libraries                   ok
Checking database user is the install user                    ok
Checking for prepared transactions                            ok
Checking for new cluster tablespace directories               ok

If pg_upgrade fails after this point, you must re-initdb the
new cluster before continuing.

Performing Upgrade
------------------
Setting locale and encoding for new cluster                   ok
Analyzing all rows in the new cluster                         ok
Freezing all rows in the new cluster                          ok
Deleting files from new pg_xact                               ok
Copying old pg_xact to new server                             ok
Setting oldest XID for new cluster                            ok
Setting next transaction ID and epoch for new cluster         ok
Deleting files from new pg_multixact/offsets                  ok
Copying old pg_multixact/offsets to new server                ok
Deleting files from new pg_multixact/members                  ok
Copying old pg_multixact/members to new server                ok
Setting next multixact ID and offset for new cluster          ok
Resetting WAL archives                                        ok
Setting frozenxid and minmxid counters in new cluster         ok
Restoring global objects in the new cluster                   ok
Restoring database schemas in the new cluster
                                                              ok
Adding ".old" suffix to old global/pg_control                 ok

If you want to start the old cluster, you will need to remove
the ".old" suffix from /db/15/global/pg_control.old.
Because "link" mode was used, the old cluster cannot be safely
started once the new cluster has been started.
Linking user relation files
                                                              ok
Setting next OID for new cluster                              ok
Sync data directory to disk                                   ok
Creating script to delete old cluster                         ok
Checking for extension updates                                notice

Your installation contains extensions that should be updated
with the ALTER EXTENSION command.  The file
    update_extensions.sql
when executed by psql by the database superuser will update
these extensions.

Upgrade Complete
----------------
Optimizer statistics are not transferred by pg_upgrade.
Once you start the new server, consider running:
    /usr/lib/postgresql/16/bin/vacuumdb --all --analyze-in-stages
Running this script will delete the old cluster's data files:
    ./delete_old_cluster.sh
timed out waiting for input: auto-logout
```

## disable old

Disable the old postgres version:

```sh
systemctl disable postgresql-15.service
```

## verify PGDATA

Verify the new `PGDATA` is correct, check with:

```sh
systemctl cat postgresql-16.service
```

Ensure the `Environment` variable is set to the new `PGDATA`:

```txt
Environment=PGDATA=/db/16
```


## start new

Start the new postgres version:

```sh
systemctl start postgresql-16.service
```

Run the `vacuumdb` command to update the optimizer statistics:

```sh
/usr/lib/postgresql/16/bin/vacuumdb --all --analyze-in-stages
```

# Patroni

If you are using `Patroni` the procedure is the same, once you have all your
extension installed, stop all nodes and work only in the primary node, you need to initialize the new data directory:

```sh
/usr/lib/postgresql/16/bin/pg_ctl initdb -D /db/16 -o --wal-segsize=16 -o --data-checksums
```

To get the current wal-segsize you can run:

```sh
ls -l /db/15/pg_wal
-rw-r----- 1 postgres postgres 2.8K Feb 25 17:24 0000003F.history
-rw-r----- 1 postgres postgres 2.8K Feb 25 17:24 00000040.history
-rw-r----- 1 postgres postgres 2.8K Feb 26 09:00 00000041.history
-rw-r----- 1 postgres postgres 2.9K Mar  2 07:52 00000042.history
-rw-r----- 1 postgres postgres  16M Mar  5 10:22 000000430000019700000055
-rw-r----- 1 postgres postgres  16M Mar  5 10:27 000000430000019700000056
-rw-r----- 1 postgres postgres  16M Mar  5 10:32 000000430000019700000057
-rw-r----- 1 postgres postgres  16M Mar  5 10:37 000000430000019700000058
```
In this case the `wal-segsize` is `16M`.

## etcd

Delete the data from etcd or remove the cluster, this is because after the
upgrade the system id may change:

```sh
patronictl remove <cluster_name>
```

> you could get this error: `CRITICAL: system ID mismatch, node <node-name> belongs to a different cluster`

## patroni.yml

After the upgrade, update `patroni.yml` and set the new `PGDATA` and bin_dir, also in the create_replica_methods, put first `basebackup` for example:

```yaml
dcs:
    postgresql:
        create_replica_methods:
            - basebackup

postgresql:
    bin_dir: /usr/lib/postgresql/16/bin
    data_dir: /db/16
```
