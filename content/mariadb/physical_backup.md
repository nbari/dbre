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

Not required but nice to have since you can save work and ensure backup can be
restored without any problems and faster

```sh
mariadb-backup --prepare --target-dir="$BACKUP_DIR"
```

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
