+++
title = "database size"
weight = 6
+++

## Database size

```sql
SELECT pg_size_pretty(pg_database_size(current_database()));
```
> this is limited to the current database


## All databases

To get size for all the databases:

```sql
SELECT
    datname AS database_name,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM
    pg_database
ORDER BY
    pg_database_size(datname) DESC;
```

## Table size

```sql
SELECT
    table_schema || '.' || table_name AS table_full_name,
    pg_size_pretty(pg_total_relation_size(table_schema || '.' || table_name)) AS total_size,
    pg_size_pretty(pg_table_size(table_schema || '.' || table_name)) AS table_size,
    pg_size_pretty(pg_indexes_size(table_schema || '.' || table_name)) AS indexes_size
FROM
    information_schema.tables
WHERE
    table_type = 'BASE TABLE'
ORDER BY
    pg_total_relation_size(table_schema || '.' || table_name) DESC;
```

## Schema not public

```sql
SELECT
    schemaname || '.' || relname AS table_full_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_table_size(relid)) AS table_size,
    pg_size_pretty(pg_indexes_size(relid)) AS indexes_size
FROM
    pg_catalog.pg_statio_user_tables
ORDER BY
    pg_total_relation_size(relid) DESC;
```
