+++
title = "Logical backup"
weight = 1
+++

Creating a logical backup:

```sh
mariadb-dump -h <host> -u <user> -p \
--events \
--routines \
--triggers \
--add-drop-database \
--compress \
--hex-blob \
--opt \
--skip-comments \
--single-transaction \
--databases <database_name> > backup.sql
```

## Restoring a logical backup:

```sh
mariadb <host> -u <user> -p < backup.sql
```
