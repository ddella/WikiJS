# On a Docker server with a container that has 'psql' installed
Execute a `psql` from a Docker container. This is executed on the Docker host but the `psql` in run inside the container.
```sh
docker exec ${CONTAINER} psql -U postgres -h localhost --command "select * from customers;"
NAME=$(date '+%Y-%m-%d %H:%M:%S')
docker exec ${CONTAINER} psql -U postgres -h 192.168.13.100 -p 5435 --command "INSERT INTO customers (firstname, date_created) VALUES ('${NAME}', CURRENT_TIMESTAMP);"
docker exec ${CONTAINER} psql -U postgres -h localhost --command "select * from customers;"
```

# On a Linux server with 'psql' installed
Execute a `psql` from a linux. The IP and port are a VIP on an HAProxy load balancer.
```sh
PGBIN="/usr/bin/psql"
PGSQL_HOST="wikijs.kloud.lan"  # VIP on HAProxy
PGSQL_PORT="5435"            # RW port
# PGSQL_PORT="5436"          # RO port
PGSQL_USERNAME="postgres"
CRT_DIR="/etc/ssl/certs"
KEY_DIR="/etc/ssl/private"

sudo ${PGBIN} "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" \
-t -c "select * from customers;"
NAME=$(date '+%Y-%m-%d %H:%M:%S')
sudo ${PGBIN} "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" \
-t -c "INSERT INTO customers (firstname, date_created) VALUES ('${NAME}', CURRENT_TIMESTAMP);"
sudo ${PGBIN} "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" \
-t -c "select * from customers;"
```

```sh
# Show the databases
sudo ${PGBIN} "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" \
-t -c "\l"
# Show the table
sudo ${PGBIN} "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" \
-t -c "\dt"
# Show the list of roles
sudo ${PGBIN} "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" \
-t -c "\du"
# Show that the table with synthetic data created when we installed PostgreSQL
sudo ${PGBIN} "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" \
-t -c "select * from customers;"
```

# Create Databse
```sh
PGBIN="/usr/bin/psql"
PGSQL_HOST="wikijs.kloud.lan" # Use the VIP for HAProxy!
PGSQL_PORT="5435"
PGSQL_USERNAME="postgres"
CRT_DIR="/etc/ssl/certs"
KEY_DIR="/etc/ssl/private"
# Create the database
sudo "${PGBIN}" "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" -t -c "CREATE DATABASE wikijs OWNER postgres;"
# Check that the database has been created
sudo "${PGBIN}" "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" -t -c "\l"
```

# Delete Database
```sh
PGBIN="/usr/bin/psql"
PGSQL_HOST="wikijs.kloud.lan" # Use the VIP for HAProxy!
PGSQL_PORT="5435"
PGSQL_USERNAME="postgres"
CRT_DIR="/etc/ssl/certs"
KEY_DIR="/etc/ssl/private"
# Check that database exists
sudo "${PGBIN}" "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" -t -c "\l"
# Delete the database
sudo "${PGBIN}" "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" -t -c "DROP DATABASE wikijs;"
# Check that the database has been created
sudo "${PGBIN}" "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" -t -c "\l"
```

sudo "${PGBIN}" "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" -t -c  "SELECT * FROM pg_stat_activity WHERE datname='wikijs';"