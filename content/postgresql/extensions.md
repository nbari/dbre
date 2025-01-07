+++
title = "Extensions"
weight = 7
+++

## Available extensions

To get a list of all available extensions, run the following command:

```sql
SELECT * FROM pg_available_extensions ORDER BY name;
```

## Installed extensions

You can list all installed extensions by running the following command:

```sql
\dx
```

Or using the following SQL query:

```sql
SELECT * FROM pg_extension;
```
