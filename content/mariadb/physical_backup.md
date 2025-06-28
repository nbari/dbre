+++
title = "Physical backup"
weight = 2
+++

## backup dir

Define a backup dir:

```sh
BACKUP_DIR="/opt/backup/backup-$(date +%Y-%m-%d_%H-%M)"
```

## Take the backup:

```sh
mariadb-backup --backup --parallel=2 --safe-slave-backup --target-dir="$BACKUP_DIR"
```

> Note: `--parallel=2` is optional, it can be adjusted based on your system's capabilities.

## prepare

```sh
mariadb-backup --prepare --target-dir="$BACKUP_DIR"
```

> Warning: Advised to run this, as it will ensure that the backup works and can be restored without any problems.

## compress

```sh
tar -cf -"${BACKUP_DIR##*/}" | zstd > "$BACKUP_DIR.tar.zst"
```

### S3

If you want to upload the backup compressed to S3, you can use the following command:

```sh
tar -cf -"${BACKUP_DIR##*/}" | s3m --pipe -x <s3 provider>/<bucket>/backup.tar
```

> Note: Replace `<s3 provider>` and `<bucket>` with your actual S3 provider and bucket name, check the [s3m documentation](https://s3m.stream/stream.html#x-compression) for more details.

## script

All together the script looks like this:

```sh
BACKUP_DIR="/opt/backup/backup-$(date +%Y-%m-%d_%H-%M)"
mariadb-backup --backup --parallel=2 --safe-slave-backup --target-dir="$BACKUP_DIR"
mariadb-backup --prepare --target-dir="$BACKUP_DIR"
tar -cf -"${BACKUP_DIR##*/}" | zstd > "$BACKUP_DIR.tar.zst"
```
