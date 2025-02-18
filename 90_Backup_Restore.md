# Backup PostgreSQL
`pg_dump` and `pg_dumpall` are primary tools in PostgreSQL’s data backup processes.

## pg_dump
`pg_dump` is a utility designed for backing up a **single** PostgreSQL database. It operates by generating a SQL script or archive file containing the database’s structure and content. `pg_dump` is ideal for scenarios where the backup of specific databases, rather than the entire PostgreSQL instance, is required.

### From a PostgreSQL container
```sh
CONTAINER="psql-01"
PGSQL_USERNAME="postgres"
DB_NAME="wikijs"
OUTPUT_FILE="output-1.sql"
docker exec ${CONTAINER} pg_dump -U ${PGSQL_USERNAME} -h localhost ${DB_NAME} > ${OUTPUT_FILE}
```

Where `wikijs` represents the name of the database to be backed up and `output.sql` is the name of the file to store the backup.

### From a server with `pg_dump`
This shows how to do a backup from an external server with the `pg_dump` utility installed:
```sh
PGBIN="/usr/bin/pg_dump"
PGSQL_HOST="wikijs.kloud.lan" # Use the VIP for HAProxy!
PGSQL_PORT="5435"
PGSQL_USERNAME="postgres"
CRT_DIR="/etc/ssl/certs"
KEY_DIR="/etc/ssl/private"
DB_NAME="wikijs"
OUTPUT_FILE="output-2.sql"
# Backup the database
sudo "${PGBIN}" "dbname=${DB_NAME} sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" -f ${OUTPUT_FILE}

```

> [!NOTE]  
> `sudo` is needed to access the TLS private key.

## pg_basebackup
Doesn't work !!!

```
pg_basebackup: error: connection to server at "wikijs.kloud.lan" (192.168.13.100), port 5435 failed: FATAL:  no pg_hba.conf entry for replication connection from host "172.18.0.1", user "postgres", SSL encryption
connection to server at "wikijs.kloud.lan" (192.168.13.100), port 5435 failed: FATAL:  no pg_hba.conf entry for replication connection from host "172.18.0.1", user "postgres", no encryption
```

```sh
PGBIN="/usr/bin/pg_basebackup"
PGSQL_HOST="wikijs.kloud.lan" # Use the VIP for HAProxy!
PGSQL_PORT="5435"
PGSQL_USERNAME="postgres"
CRT_DIR="/etc/ssl/certs"
KEY_DIR="/etc/ssl/private"
DB_NAME="wikijs"
OUTPUT_FILE="output-2.sql"

PGSSLCERT="${CRT_DIR}/psql.crt"
PGSSLKEY="${KEY_DIR}/psql.key"
PGSSLROOTCERT="${CRT_DIR}/wikijs-ca.crt"
PGSSLMODE="verify-ca"
# Tar and Compressed Format
sslmode=${PGSSLMODE} sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key sudo "${PGBIN}" -h ${PGSQL_HOST} -U ${PGSQL_USERNAME} -p ${PGSQL_PORT} -D /tmp/ -Ft -z -Xs -P
# Plain Format
sslmode=${PGSSLMODE} sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key sudo "${PGBIN}" -h ${PGSQL_HOST} -U ${PGSQL_USERNAME} -p ${PGSQL_PORT} -D /tmp/ -Fp -Xs -P
```

```sh
CONTAINER="psql-01"
PGSQL_USERNAME="postgres"
docker exec ${CONTAINER} pg_basebackup -U ${PGSQL_USERNAME} -h localhost -D /bitnami/postgresql/tmp/ -Fp -Xs -P
```


https://www.postgresql.org/docs/current/libpq-envars.html
https://www.postgresql.org/docs/current/app-pgbasebackup.html

## pg_dumpall
In contrast, `pg_dumpall` is a tool used for creating a backup of an entire PostgreSQL instance. This includes all databases, along with global objects such as roles and tablespaces. `pg_dumpall` generates an SQL script that, when executed, recreates the entire database cluster. This tool is particularly useful for comprehensive backups, where preservation of the complete database environment is essential.


# pg_hba.conf
```sh
cat <<EOF | sudo tee /opt/psql/postgresql/psql_conf/pg_hba.conf > /dev/null
# Allow any user on the local system to connect to any database with
# any database user name using Unix-domain sockets (the default for local
# connections).
# https://www.postgresql.org/docs/17/auth-pg-hba-conf.html
# https://www.postgresql.org/docs/current/app-pgbasebackup.html
# https://www.postgresql.org/docs/current/libpq-envars.html
#
# TYPE    DATABASE       USER        CIDR-ADDRESS    METHOD
local     all             all                        trust
hostssl   replication    postgres    samehost        cert
hostssl   replication    postgres    0.0.0.0/0       cert
hostssl   replication    postgres    ::/0            cert
hostssl   all            all         0.0.0.0/0       cert
hostssl   all            all         ::/0            cert
host      all            repmgr      0.0.0.0/0       trust
host      repmgr         repmgr      0.0.0.0/0       trust
host      repmgr         repmgr      ::/0            trust
host      all            all         0.0.0.0/0       trust
host      all            all         ::/0            trust
local     all            all                         trust
EOF
sudo chown 1001:0 /opt/psql/postgresql/conf/pg_hba.conf
docker exec ${CONTAINER} psql -U postgres -h localhost --command "SELECT pg_reload_conf();"
```

```sh
CONTAINER="psql-02"
# Get version
docker exec ${CONTAINER} psql -U postgres -h localhost --command "SELECT pg_reload_conf();"
```

host     all             repmgr    0.0.0.0/0    trust
host     repmgr          repmgr    0.0.0.0/0    trust
host     repmgr          repmgr    ::/0         trust
hostssl     replication  all       0.0.0.0/0    cert
hostssl     replication  all       ::/0         cert
hostssl     all          all       0.0.0.0/0    cert
hostssl     all          all       ::/0         cert
host     all             all       0.0.0.0/0    trust
host     all             all       ::/0         trust
local    all             all                    trust
