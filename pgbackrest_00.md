# pgBackRest

Create a Podman network and 6 containers with:

```bash
podman network create pgbackrest_network

PGSTDBY='pgnode2 pgnode3'

PGNODES="pgnode1 ${PGSTDBY} pgrestore"

BKPSERVERS='pgbkp1 pgbkp2'

ALLSERVERS="${BKPSERVERS} ${PGNODES}"

podman container rm -f ${ALLSERVERS}

for i in ${ALLSERVERS}
do

podman container run \
    -itd --network=pgbackrest_network --name=${i} --hostname ${i}.my-domain debian

done
```

# Environment variables

```bash
# Enter the major version to be installed
read -p 'Enter the major version to be installed: ' PGVERSION

PGDATA="/var/lib/postgresql/${PGVERSION}/main"
PGCONF="/etc/postgresql/${PGVERSION}/main"
PGBIN="/usr/lib/postgresql/${PGVERSION}/bin"
```

Environment variables to a file:
```bash
cat << EOF > env
PGVERSION='${PGVERSION}'
PGDATA='${PGDATA}'
PGCONF='${PGCONF}'
PGBIN='${PGBIN}'
PGSTDBY='${PGSTDBY}'
PGNODES='${PGNODES}'
BKPSERVERS='${BKPSERVERS}'
ALLSERVERS='${ALLSERVERS}'
EOF
```

# Installation script

```bash
cat << EOF > /tmp/pginstall.sh
apt update
apt install -y postgresql-common
/usr/share/postgresql-common/pgdg/apt.postgresql.org.sh -y
apt -y install postgresql-${PGVERSION} pgbackrest ssh finger cron
apt clean

# Symbolic links to all configuration files
mv ${PGCONF}/* ${PGDATA}/
ls -1 ${PGDATA}/*conf* | grep -Ev '^$' | tr -d ':' | \
    xargs -i ln -s {} ${PGCONF}/

# listen_addresses = '*'
sed "s/\(#listen_addresses.*\)/\1\nlisten_addresses = '*'/g" -i \
    ${PGDATA}/postgresql.conf

# pg_hba.conf changes
echo "
host all all all scram-sha-256
host replication all all scram-sha-256
" >> ${PGDATA}/pg_hba.conf

# Copy all files from /etc/skel to postgres home directory
cp -v /etc/skel/.* ~postgres/

# Setting some environment variables on profile file inside postgres
# user home directory
echo "
export PATH=\"\${PATH}:${PGBIN}\"
export PGDATA='${PGDATA}'
" >> ~postgres/.bashrc

# .pgpass
echo '*:*:*:user_pgbckrst:123' > ~postgres/.pgpass
chmod 0600 ~postgres/.pgpass

# Change ownership recursively to user and group postgres
chown -R postgres: /{var/lib,etc}/postgresql

# Start SSH service
/etc/init.d/ssh start

EOF
```

# Replica script

```bash
cat << EOF > /tmp/pg_build_replica && chmod +x /tmp/pg_build_replica
#!/bin/bash

# First parameter as primary server
PRI="\${1}"

# Remove the data directory
rm -fr ${PGDATA}

# pg_basebackup
DBCONN="host=\${PRI} dbname=postgres user=user_pgbckrst password=123"
${PGBIN}/pg_basebackup -c fast \
    -D ${PGDATA} -Fp -Xs -P -R -CS rs_\`hostname -s\` -d "\${DBCONN}"

# Start PostgreSQL service
${PGBIN}/pg_ctl -D ${PGDATA} start

EOF
```


# Packages installation

```bash
for i in ${ALLSERVERS}
do

echo "${i} ======================="
cat /tmp/pginstall.sh | podman container exec -i ${i} /bin/bash
podman container cp /tmp/pg_build_replica ${i}:/usr/local/bin

done
```


# Start PostgreSQL service on master node

```bash
podman container exec pgnode1 /etc/init.d/postgresql start
```



# Backrest user

```bash
# SQL String to create the Backrest and replication users
SQL="
CREATE ROLE user_pgbckrst
    REPLICATION
    LOGIN
    ENCRYPTED PASSWORD '123'

CREATE ROLE user_rep
    LOGIN
    REPLICATION
    ENCRYPTED PASSWORD '123';    
"

# Execute the SQL command
echo ${SQL} | podman container exec -iu postgres pgnode1 psql
```


# Create the replicas

```bash
for i in ${PGSTDBY}
do

podman container exec -iu postgres ${i} \
    /usr/local/bin/pg_build_replica pgnode1

done
```

# Confirm if the standbys are attached to the primary

```bash
podman exec -u postgres pgnode1 psql -xc 'SELECT * FROM pg_stat_replication'
```
```
-[ RECORD 1 ]----+------------------------------
pid              | 6619
usesysid         | 16384
usename          | user_pgbckrst
application_name | 16/main
client_addr      | 10.89.1.19
client_hostname  |
client_port      | 39264
backend_start    | 2024-07-17 22:55:17.818855+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/4041438
write_lsn        | 0/4041438
flush_lsn        | 0/4041438
replay_lsn       | 0/4041438
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2024-07-17 23:14:13.391702+00
-[ RECORD 2 ]----+------------------------------
pid              | 6622
usesysid         | 16384
usename          | user_pgbckrst
application_name | 16/main
client_addr      | 10.89.1.20
client_hostname  |
client_port      | 43862
backend_start    | 2024-07-17 22:55:19.065912+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/4041438
write_lsn        | 0/4041438
flush_lsn        | 0/4041438
replay_lsn       | 0/4041438
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2024-07-17 23:14:13.39169+00
```




# Confirm replication

```bash
# Test database creation
podman exec -u postgres pgnode1 psql -qc 'CREATE DATABASE db_test'

# Test table creation
podman exec -u postgres pgnode1 psql -qc \
    'CREATE TABLE tb_test AS SELECT generate_series(1, 5) AS c1' db_test

# Check the test table on replicas
podman exec -u postgres pgnode2 psql -c 'SELECT c1 FROM tb_test' db_test
podman exec -u postgres pgnode3 psql -c 'SELECT c1 FROM tb_test' db_test
```
```
 c1
----
  1
  2
  3
  4
  5
(5 rows)
```

# Check pgBackRest version

```bash
for i in ${ALLSERVERS}
do
echo "${i}:"
podman container exec -itu postgres ${i} pgbackrest version

done
```
```
pgBackRest 2.52.1
```
