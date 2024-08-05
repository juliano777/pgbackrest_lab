# 

SETTING UP STREAMING REPLICATION USING PGBACKREST

For my configuration as seen earlier, i have three server as seen below..

pgnode1 primary
pgnode2 standby1
pgnode3 stadnby2
pgbkp1  pgbackrest server


```bash
for i in ${PGSTDBY}
do
# ============================================================================

podman container exec -iu postgres ${i} bash -c \
"cat << EOF > /etc/pgbackrest.conf
[global]
repo1-host=pgbkp1
log-level-console=info
repo1-host-user=postgres
log-level-file=debug
delta=y

[demo]
pg1-path=${PGDATA}
pg1-user=user_pgbckrst
pg1-port=5432
recovery-option=primary_conninfo=host=pgnode1 user=user_rep port=5432
EOF"

# ============================================================================
done
```




```bash
for i in ${PGSTDBY}
do

podman container exec -iu postgres ${i} \
    pgbackrest --stanza=demo --type=standby --no-delta restore

done
```

