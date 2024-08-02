# pgBackRest

```bash
# Enter the major version to be installed
read -p 'Enter the major version to be installed: ' PGVERSION
PGDATA="/var/lib/postgresql/${PGVERSION}/main"
PGCONF="/etc/postgresql/${PGVERSION}/main"
```


# Update the pgbkp1 pgBackRest configuration file

```bash
podman container exec -iu postgres pgbkp1 \
bash -c "cat << EOF > /etc/pgbackrest.conf
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
log-level-console=info
log-level-file=debug
start-fast=y
delta=y

[demo]
pg1-host=pgnode1
pg1-host-user=postgres
pg1-path=${PGDATA}
pg1-port=5432
EOF"
```


# Update the primary pgBackRest configuration file

``` bash

podman container exec -iu postgres pgnode1 bash -c "cat << EOF > /etc/pgbackrest.conf
[global]
repo1-host=pgbkp1
log-level-console=info
repo1-host-user=postgres
log-level-file=debug
delta=y

[demo]
pg1-path=${PGDATA}
pg1-user=postgres
pg1-port=5432
EOF"
```

# Configure archiving for Postgres
``` bash
podman container exec -iu postgres pgnode1 bash -c \
"cat << EOF > ${PGCONF}/conf.d/00_archiving.conf
archive_mode = on
archive_command = 'pgbackrest --stanza=demo archive-push %p'
logging_collector = on
EOF"
```

# Restart PostgreSQL service on master node

```bash
podman container exec pgnode1 /etc/init.d/postgresql restart
```


# Create the stanza and check the configuration on the pgbkp1

```bash
podman container exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo stanza-create
```

# First backup on the backup server from the primary server
```bash
podman exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo backup --type=full

```

# The info command can be executed from any server where pgBackRest is correctly configured

```bash
podman exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo info
```
```
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (16): 000000010000000000000023/000000010000000000000027

        full backup: 20240725-130425F
            timestamp start/stop: 2024-07-25 13:04:25+00 / 2024-07-25 13:04:30+00
            wal start/stop: 000000010000000000000027 / 000000010000000000000027
            database size: 22.2MB, database backup size: 22.2MB
            repo1: backup set size: 2.9MB, backup size: 2.9MB
```

# Run a check on the stanza to make sure everything is Ok
```bash
podman exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo check
```
```
2024-07-25 13:05:55.835 P00   INFO: check command begin 2.52.1: --exec-id=8854-71949e6f --log-level-console=info --log-level-file=debug --pg1-host=pgnode1 --pg1-host-user=postgres --pg1-path=/var/lib/postgresql/16/main --pg1-port=5432 --repo1-path=/var/lib/pgbackrest --stanza=demo
2024-07-25 13:05:56.724 P00   INFO: check repo1 configuration (primary)
2024-07-25 13:05:56.927 P00   INFO: check repo1 archive for WAL (primary)
2024-07-25 13:05:57.529 P00   INFO: WAL segment 000000010000000000000028 successfully archived to '/var/lib/pgbackrest/archive/demo/16-1/0000000100000000/000000010000000000000028-3c7160ac5c757c2e9ef1f208e2e6d5c779bc0487.gz' on repo1
2024-07-25 13:05:57.630 P00   INFO: check command end: completed successfully (1797ms)
```

# Take more two backups with the following command (repeat it two times)
```bash
podman exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo backup --type=full
```

# The info command can be executed from any server where pgBackRest is correctly configured

```bash
podman exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo info
```
```
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (16): 00000001000000000000002A/00000001000000000000002C

        full backup: 20240725-130626F
            timestamp start/stop: 2024-07-25 13:06:26+00 / 2024-07-25 13:06:31+00
            wal start/stop: 00000001000000000000002A / 00000001000000000000002A
            database size: 22.2MB, database backup size: 22.2MB
            repo1: backup set size: 2.9MB, backup size: 2.9MB

        full backup: 20240725-130637F
            timestamp start/stop: 2024-07-25 13:06:37+00 / 2024-07-25 13:06:42+00
            wal start/stop: 00000001000000000000002C / 00000001000000000000002C
            database size: 22.2MB, database backup size: 22.2MB
            repo1: backup set size: 2.9MB, backup size: 2.9MB
```

Notice that the first backup was deleted due to policy to retention backup (2 backups only).
So automatically the oldest one will be removed.


# Remove the oldest backup

```bash
# Get the id of the oldest backup
OLDEST=`podman exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo info | grep 'full backup:' | head -1 | cut -f2 -d: | xargs`

# Remove the oldest backup
podman exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo expire --set=${OLDEST}

# Check backups
podman exec -iu postgres pgbkp1 \
    pgbackrest --stanza=demo info
```
```
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (16): 00000001000000000000002A/00000001000000000000002C

        full backup: 20240725-130637F
            timestamp start/stop: 2024-07-25 13:06:37+00 / 2024-07-25 13:06:42+00
            wal start/stop: 00000001000000000000002C / 00000001000000000000002C
            database size: 22.2MB, database backup size: 22.2MB
            repo1: backup set size: 2.9MB, backup size: 2.9MB
```

One backup was deleted, the oldest one and then more storage space was freed up.

