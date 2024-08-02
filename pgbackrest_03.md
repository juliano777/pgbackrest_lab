# Backups full, differential and incremental


#  Change the pgbackrest.conf file
```bash
podman container exec -iu postgres pgbkp1 \
bash -c "cat << EOF > /etc/pgbackrest.conf
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
repo1-retention-full-type=count
repo1-retention-diff=1
log-level-file=debug
log-level-console=info
start-fast=y
delta=y

[demo]
pg1-host=pgnode1
pg1-path=${PGDATA}
EOF"
```


We’ll keep 2 full and 1 differential backup.

# Full backup
``` bash
# Manually remove all backups
podman container exec -iu postgres pgbkp1 \
    rm -fr /var/lib/pgbackrest/backup/demo/2*

#
podman container exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo expire
```


Let’s take one backup of each kind (full/diff/incr):

# Full backup
``` bash
podman container exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo backup --type=full
```

# Differential backup

```bash
podman container exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo backup --type=diff
```

# Incremental backup
```bash
podman container exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo backup --type=incr
```

# List backups
```bash
podman container exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo info
```
```
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (16): 00000001000000000000002A/000000010000000000000032

        full backup: 20240725-144329F
            timestamp start/stop: 2024-07-25 14:43:29+00 / 2024-07-25 14:43:34+00
            wal start/stop: 00000001000000000000002E / 00000001000000000000002E
            database size: 22.2MB, database backup size: 22.2MB
            repo1: backup set size: 2.9MB, backup size: 2.9MB

        diff backup: 20240725-144329F_20240725-144345D
            timestamp start/stop: 2024-07-25 14:43:45+00 / 2024-07-25 14:43:47+00
            wal start/stop: 000000010000000000000030 / 000000010000000000000030
            database size: 22.2MB, database backup size: 133.2KB
            repo1: backup set size: 2.9MB, backup size: 8.2KB
            backup reference list: 20240725-144329F

        incr backup: 20240725-144329F_20240725-144400I
            timestamp start/stop: 2024-07-25 14:44:00+00 / 2024-07-25 14:44:03+00
            wal start/stop: 000000010000000000000032 / 000000010000000000000032
            database size: 22.2MB, database backup size: 135.0KB
            repo1: backup set size: 2.9MB, backup size: 8.3KB
            backup reference list: 20240725-144329F, 20240725-144329F_20240725-144345D
```


# Let’s take a new differential backup to observe the expiration

```bash
podman container exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo backup --type=diff
```

# List backups
```bash
podman container exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo info
```
```
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (16): 00000001000000000000002A/000000010000000000000034

        full backup: 20240725-144329F
            timestamp start/stop: 2024-07-25 14:43:29+00 / 2024-07-25 14:43:34+00
            wal start/stop: 00000001000000000000002E / 00000001000000000000002E
            database size: 22.2MB, database backup size: 22.2MB
            repo1: backup set size: 2.9MB, backup size: 2.9MB

        diff backup: 20240725-144329F_20240725-144345D
            timestamp start/stop: 2024-07-25 14:43:45+00 / 2024-07-25 14:43:47+00
            wal start/stop: 000000010000000000000030 / 000000010000000000000030
            database size: 22.2MB, database backup size: 133.2KB
            repo1: backup set size: 2.9MB, backup size: 8.2KB
            backup reference list: 20240725-144329F

        incr backup: 20240725-144329F_20240725-144400I
            timestamp start/stop: 2024-07-25 14:44:00+00 / 2024-07-25 14:44:03+00
            wal start/stop: 000000010000000000000032 / 000000010000000000000032
            database size: 22.2MB, database backup size: 135.0KB
            repo1: backup set size: 2.9MB, backup size: 8.3KB
            backup reference list: 20240725-144329F, 20240725-144329F_20240725-144345D

        diff backup: 20240725-144329F_20240725-145048D
            timestamp start/stop: 2024-07-25 14:50:48+00 / 2024-07-25 14:50:51+00
            wal start/stop: 000000010000000000000034 / 000000010000000000000034
            database size: 22.2MB, database backup size: 137KB
            repo1: backup set size: 2.9MB, backup size: 8.5KB
            backup reference list: 20240725-144329F
```



As we can see, the oldest WAL archive in the repository is
00000001000000000000002A, while the oldest start WAL backup location is
00000001000000000000002E.
When a differential backup expires, all incremental backups associated with
the differential backup will also expire. When not defined all differential
backups will be kept until the full backups they depend on expire.


# Full backup (2 times)
``` bash
podman container exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo backup --type=full
```

# List backups
```bash
podman container exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo info
```
```
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (16): 000000010000000000000036/000000010000000000000038

        full backup: 20240725-175010F
            timestamp start/stop: 2024-07-25 17:50:10+00 / 2024-07-25 17:50:15+00
            wal start/stop: 000000010000000000000036 / 000000010000000000000036
            database size: 22.2MB, database backup size: 22.2MB
            repo1: backup set size: 2.9MB, backup size: 2.9MB

        full backup: 20240725-175109F
            timestamp start/stop: 2024-07-25 17:51:09+00 / 2024-07-25 17:51:12+00
            wal start/stop: 000000010000000000000038 / 000000010000000000000038
            database size: 22.2MB, database backup size: 22.2MB
            repo1: backup set size: 2.9MB, backup size: 2.9MB
```

The WAL archives before 00000001000000000000002E have been removed.
Also, we can remark that the differential backup have been expired too:

```

. . .

2024-07-25 17:51:13.516 P00   INFO: repo1: expire full backup set 20240725-144329F, 20240725-144329F_20240725-144345D, 20240725-144329F_20240725-144400I, 20240725-144329F_20240725-145048D
2024-07-25 17:51:13.518 P00   INFO: repo1: remove expired backup 20240725-144329F_20240725-145048D
2024-07-25 17:51:13.519 P00   INFO: repo1: remove expired backup 20240725-144329F_20240725-144400I
2024-07-25 17:51:13.519 P00   INFO: repo1: remove expired backup 20240725-144329F_20240725-144345D
2024-07-25 17:51:13.519 P00   INFO: repo1: remove expired backup 20240725-144329F
2024-07-25 17:51:13.545 P00   INFO: repo1: 16-1 remove archive, start = 00000001000000000000002E, stop = 000000010000000000000035
2024-07-25 17:51:13.545 P00   INFO: expire command end: completed successfully (31ms)

```



