# SSH keys interchange between the nodes


# Start nodes if they aren't

```bash
PGSTDBY='pgnode2 pgnode3'

PGNODES="pgnode1 ${PGSTDBY} pgrestore"

BKPSERVERS='pgbkp1 pgbkp2'

ALLSERVERS="${BKPSERVERS} ${PGNODES}"

podman container start ${ALLSERVERS}
```

# Generate all SSH keys for postgres user
```bash

for i in ${ALLSERVERS}
do

PGHOME=`podman container exec -iu postgres pgnode1 finger postgres | \
    grep 'Directory' | awk '{print $2}'`

podman container exec -iu postgres ${i} \
    ssh-keygen -P '' -t rsa -f ${PGHOME}/.ssh/id_rsa

podman container exec -iu postgres ${i} \
    cat ${PGHOME}/.ssh/id_rsa.pub >> /tmp/authorized_keys

done
```

# Create ~postgres/.ssh/authorized_keys
```bash

for i in ${ALLSERVERS}
do

PGHOME=`podman container exec -iu postgres ${i} finger postgres | \
    grep 'Directory' | awk '{print $2}'`

cat /tmp/authorized_keys | \
    podman exec -iu postgres ${i} \
    bash -c "cat > ${PGHOME}/.ssh/authorized_keys"

done
```


# Add all servers as known hosts
```bash

for i in ${ALLSERVERS}
do
    PGHOME=`podman container exec -iu postgres ${i} finger postgres | \
    grep 'Directory' | awk '{print $2}'`

    for j in ${ALLSERVERS}
    do
        echo "${i} -> ${j}"
        podman container exec -iu postgres ${i} \
        bash -c "ssh-keyscan -H ${j} >> ${PGHOME}/.ssh/known_hosts"
    done
done
```


