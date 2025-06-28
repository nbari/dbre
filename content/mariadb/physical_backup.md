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

## prepare

```sh
mariadb-backup --prepare --target-dir="$BACKUP_DIR"
```

> Warning: Advised to run this, as it will ensure that the backup works and can be restored without any problems.

## compress

```sh
tar -cf -"${BACKUP_DIR##*/}" | zstd > "$BACKUP_DIR.tar.zst"
```

## script

All together the script looks like this:

```sh
BACKUP_DIR="/opt/backup/backup-$(date +%Y-%m-%d_%H-%M)"
mariadb-backup --backup --parallel=2 --safe-slave-backup --target-dir="$BACKUP_DIR"
mariadb-backup --prepare --target-dir="$BACKUP_DIR"
tar -cf -"${BACKUP_DIR##*/}" | zstd > "$BACKUP_DIR.tar.zst"
```
