# Steps to Deploy PostgreSQL High Availability

## Option A. Rebuild the image (Optional)
I wanted to rebuild the image because 'Bitnami' didn't create a user associated with `UID:1001`. It's not a big of an issue but I wanted to have the users especially if you're doing mTLS. The only change in the new image is the creation of user `postgres` with `UID:1001` and `GID:0`.
- Creating a `Dockerfile` with the base image from the PostgreSQL one
- Add the new users
- Rebuild the image

```sh
cat <<EOF >psql_Dockerfile
# https://github.com/bitnami/containers/tree/main/bitnami/postgresql-repmgr/17/debian-12
# https://hub.docker.com/r/bitnami/postgresql-repmgr
#
# docker build -t postgresql-repmgr:17.2.0-debian-12-r1-patch1 .
#
FROM docker.io/bitnami/postgresql-repmgr:17.2.0-debian-12-r1
USER 0
RUN useradd -u 1001 -g 0 postgres
USER 1001
ENTRYPOINT [ "/opt/bitnami/scripts/postgresql-repmgr/entrypoint.sh" ]
CMD [ "/opt/bitnami/scripts/postgresql-repmgr/run.sh" ]
EOF
# Build the image
docker build -t bitnami/postgresql-repmgr:17.2.0-debian-12-r1-patch1 -f psql_Dockerfile .
```

## Option B. Another method (Optional)
> [!IMPORTANT]  
> Don't do `Option B` if you did `option A` above.

You could rebuild from the original repository:

- Clone only the repo that we need, which is `bitnami/postgresql-repmgr/17/debian-12`.
- Edit the `Dockerfile` to add the users `postgres` with `UID:1001`.
- Rebuild the image

```sh
DIR=${PWD}
# Clone just the repo neededed
git clone -n --depth=1 --filter=tree:0 https://github.com/bitnami/containers.git
cd containers
git sparse-checkout set --no-cone /bitnami/postgresql-repmgr/17/debian-12/
git checkout
cd bitnami/postgresql-repmgr/17/debian-12/
# Add the user 'postgres' to the Dockerfile
sed -i 's/USER 1001/RUN useradd -u 1001 -g 0 postgres\nUSER 1001/' Dockerfile
# Rebuild the image
docker build -t bitnami/postgresql-repmgr:17.2.0-debian-12-r1-patch1 .
# Going back
cd ${DIR}
# Cleanup the repo we just cloned
rm -rf containers
```

# Create PRIMARY and STANDBY PostgreSQL
Create a primary and standby PostgreSQL container with `pgAdmin`. The later one is optional.

- Create a local directory on the Docker hosts to map the PostgreSQL container
- Create a local directory on the Docker hosts to map pgAdmin container
- Copy the TLS certificates/private key
  - The PostgreSQL certificate/private key
  - The RootCA certificate
- Set the proper permissions on every directory

> [!IMPORTANT]  
> As this is a non-root container, the mounted files and directories must have the proper permissions.  
> PostgreSQL: `UID:1001`  
> pgAdmin: `UID:5050`

Do this on both Docker hosts.
```sh
# Create directories
sudo mkdir -p /opt/psql/postgresql/certs
sudo mkdir -p /opt/psql/postgresql/tmp
sudo mkdir -p /opt/psql/postgresql/logs
sudo mkdir -p /opt/psql/postgresql/psql_conf
sudo mkdir -p /opt/pgadmin
# For repmgr
sudo mkdir -p /opt/repmgr/data
sudo mkdir -p /opt/repmgr/conf/
# Copy the certs. Every PostgreSQL servers will use the same certs
sudo cp psql.key /opt/psql/postgresql/certs/.
sudo cp psql.crt /opt/psql/postgresql/certs/.
sudo cp wikijs-ca.crt /opt/psql/postgresql/certs/.
# Set owner
sudo chown 1001:1001 -R /opt/psql/postgresql
sudo chown 1001:1001 -R /opt/repmgr
sudo chown 5050:5050 -R /opt/pgadmin
```

Create the Docker Compose YAML file to start both containers. This has to be exectuded on all Docker hosts.

You need to uncomment the right section depending on where you execute the commands.
```sh
# PRIMARY PostgreSQL
# HOST="psql-01"
# IPv4="192.168.255.21"
# REPMGR_NODE_ID="1"
# REPMGR_NODE_PRIORITY="100"
# PGADMIN_HOST="pgadmin-01"
# PGADMIN_IPv4="192.168.255.31"

# STANDBY PostgreSQL
# HOST="psql-02"
# IPv4="192.168.255.22"
# REPMGR_NODE_ID="2"
# REPMGR_NODE_PRIORITY="100"
# PGADMIN_HOST="pgadmin-02"
# PGADMIN_IPv4="192.168.255.32"

# Same for all PostgreSQL
REPMGR_PARTNER_NODES="psql-01,psql-02"
REPMGR_PRIMARY_HOST="psql-01"
YAML_FILE="docker.yaml"

cat <<EOF >${YAML_FILE}
# Start the deployment
#   docker compose --file ${YAML_FILE} up --detach
#   docker compose --file ${YAML_FILE} restart
# Destroy the deployment
#   docker compose --file ${YAML_FILE} down --remove-orphans
# Stop by keeps everything
#   docker compose --file ${YAML_FILE} stop
#   docker compose --file ${YAML_FILE} rm --force
# List the running container
#   docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Status}}\t{{.Names}}'
# Jump inside a container
#   docker exec -it ${HOST} /bin/sh
# Check logs
#   docker compose --file ${YAML_FILE} logs
networks:
  wikijs:
    name: core-net
    external: true

services:
# ------------------------------------------------#
#        PostgreSQL Container
# ------------------------------------------------#
  postgresql:
    container_name: ${HOST}
    hostname: ${HOST}
    domainname: kloud.lan
    image: 'bitnami/postgresql-repmgr:17.2.0-debian-12-r1-patch1'
    environment:
      # - POSTGRES_USER=postgresadmin
      - POSTGRESQL_PASSWORD=custompassword
      # - REPMGR_USERNAME=repmgrusername
      - REPMGR_PASSWORD=repmgrpassword
      - REPMGR_NODE_NAME=${HOST}
      - REPMGR_PRIMARY_HOST=${REPMGR_PRIMARY_HOST}
      - REPMGR_NODE_NETWORK_NAME=${HOST}
      - REPMGR_PARTNER_NODES=${REPMGR_PARTNER_NODES}
      - REPMGR_NODE_ID=${REPMGR_NODE_ID}
      - REPMGR_NODE_PRIORITY=${REPMGR_NODE_PRIORITY}
      - REPMGR_SWITCH_ROLE=yes
      # - REPMGR_LOG_LEVEL=DEBUG
      - POSTGRESQL_ENABLE_TLS=yes
      - POSTGRESQL_TLS_CERT_FILE=/bitnami/postgresql/certs/psql.crt
      - POSTGRESQL_TLS_KEY_FILE=/bitnami/postgresql/certs/psql.key
      - POSTGRESQL_TLS_CA_FILE=/bitnami/postgresql/certs/wikijs-ca.crt
      - POSTGRESQL_TIMEZONE=America/Montreal
      - POSTGRESQL_LOG_TIMEZONE=America/Montreal
      - TZ=America/Montreal
      - PGUSER=postgres
      - PGSSLMODE=verify-ca
      - PGSSLCERT=/bitnami/postgresql/certs/psql.crt
      - PGSSLKEY=/bitnami/postgresql/certs/psql.key
      - PGSSLROOTCERT=/bitnami/postgresql/certs/wikijs-ca.crt
    ports:
    # You need to expose the port outside the container, for the health check by the load balancer
      - '5432:5432'
    # logging:
    #   driver: "none"
    restart: unless-stopped
    volumes:
      - /opt/psql/postgresql:/bitnami/postgresql
      - /opt/psql/postgresql/tmp:/opt/bitnami/postgresql/tmp
      - /opt/psql/postgresql/logs:/opt/bitnami/postgresql/logs
      # postgresql.conf & pg_hba.conf
      - /opt/psql/postgresql/psql_conf:/opt/bitnami/postgresql/conf
      # repmgr directories
      - /opt/repmgr/data:/bitnami/postgresql/data
      - /opt/repmgr/conf:/opt/bitnami/repmgr/conf
    networks:
      wikijs:
        # Define a static ip for the container. The containter can be accessible by others devices on the LAN network with this IP.
        ipv4_address: ${IPv4}

# ------------------------------------------------#
#        pgAdmin Container
# ------------------------------------------------#
  pgadmin:
    container_name: ${PGADMIN_HOST}
    hostname: ${PGADMIN_HOST}
    domainname: kloud.lan
    image: dpage/pgadmin4:8.13.0
    environment:
      - PGADMIN_DEFAULT_EMAIL='testpg@kloud.lan'
      - PGADMIN_DEFAULT_PASSWORD='testpg'
      - PGADMIN_LISTEN_PORT=8080
      - PGADMIN_CONFIG_LOGIN_BANNER="Authorised users only! username = testpg at kloud.lan - password = testpg"
      - TZ=America/Montreal
    restart: unless-stopped
    depends_on:
      - postgresql
    # ports:
    # No need to expose the port outside the container, we're using a load balancer
    #   - 8080:8080
    volumes:
      - /opt/pgadmin:/var/lib/pgadmin
    networks:
      wikijs:
        # Define a static ip for the container.
        # The containter can be accessible by others devices on the LAN network with this IP.
        ipv4_address: ${PGADMIN_IPv4}
EOF
```

Start the containers and check that they all run, with the commands:
```sh
# Start the containers
docker compose --file ${YAML_FILE} up --detach
# Check the logs
docker compose --file ${YAML_FILE} logs -f
# List the containers
# docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Status}}\t{{.Names}}'
```

# PostgreSQL - Primary
Run the commands below on the Primary PostgreSQL server to:
- Verify the PostgreSQL cluster status
- Create a dummy table
- Create synthetic entries inside the table created in the step above

```sh
CONTAINER="psql-01"
# Get version
docker exec ${CONTAINER} psql -U postgres -h localhost --command "SELECT version();"
# Check the cluster's status
docker exec ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf cluster show
docker exec ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf daemon status
# Execute a "psql" command inside the container in a one-liner
docker exec ${CONTAINER} psql -U postgres -h localhost --command "CREATE TABLE customers (firstname text, customer_id serial, date_created timestamp);"
# Show configuration file
docker exec ${CONTAINER} psql -U postgres -h localhost --command "show config_file;"
# Show pg_hba.conf file
docker exec ${CONTAINER} psql -U postgres -h localhost --command "show hba_file;"
# Show where the database files are stored
docker exec ${CONTAINER} psql -U postgres -h localhost --command "show data_directory;"
# Show the databases
docker exec ${CONTAINER} psql -U postgres -h localhost --command "\l"
# Show the table
docker exec ${CONTAINER} psql -U postgres -h localhost --command "\dt"
# Show the list of roles
docker exec ${CONTAINER} psql -U postgres -h localhost --command "\du"
# Show that the table is empty
docker exec ${CONTAINER} psql -U postgres -h localhost --command "select * from customers;"
# Insert data
docker exec ${CONTAINER} psql -U postgres -h localhost --command "INSERT INTO customers (firstname, date_created) VALUES ('Daniel', CURRENT_TIMESTAMP);"
docker exec ${CONTAINER} psql -U postgres -h localhost --command "INSERT INTO customers (firstname, date_created) VALUES ('Nathalie', CURRENT_TIMESTAMP);"
docker exec ${CONTAINER} psql -U postgres -h localhost --command "INSERT INTO customers (firstname, date_created) VALUES ('Anmarie', CURRENT_TIMESTAMP);"
docker exec ${CONTAINER} psql -U postgres -h localhost --command "INSERT INTO customers (firstname, date_created) VALUES ('Jessica', CURRENT_TIMESTAMP);"
# Show that there's records. Go to the standby and excute the same command. The table should be identical
docker exec ${CONTAINER} psql -U postgres -h localhost --command "select * from customers;"
# Delete data. Go to the Standby and the record shouldn't be there
# docker exec ${CONTAINER} psql -U postgres -h localhost --command "DELETE FROM customers WHERE firstname = 'Daniel';"
```

CONTAINER="psql-01"
# Get version
docker exec ${CONTAINER} psql -U postgres -h localhost --command "SELECT pg_reload_conf();"

# PostgreSQL - Standby
Run the commands below on the Primary PostgreSQL server to:
- Verify the PostgreSQL cluster status
- Readsthe table created on the primary

```sh
CONTAINER="psql-02"
# Check the cluster's status
docker exec ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf cluster show
docker exec ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf daemon status
# Execute a "psql" command inside the container in a one-liner
docker exec ${CONTAINER} psql -U postgres -h localhost --command "select * from customers;"
```

```
# --------------------------------------------------------------------------------
#     !!!! ***** Congrats 🎉🎉🎉🥳🥳🥳🍾🍾🍾 You have a PostgreSQL Cluster***** !!!!!
# --------------------------------------------------------------------------------
```

# References
[High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html)  
[Bitnami package for PostgreSQL HA](https://bitnami.com/stack/postgresql-ha/containers)  
[Bitnami PostgreSQL H.A. on GitHub](https://github.com/bitnami/containers/tree/main/bitnami/postgresql-repmgr)  
https://github.com/bitnami/containers/blob/main/bitnami/postgresql-repmgr/README.md
[Test with data](https://vuyisile.com/high-availability-in-postgresql-replication-with-docker/)  
[Dockerfile for PostgreSQL 17 on Debian 12](https://github.com/bitnami/containers/blob/main/bitnami/postgresql/17/debian-12/Dockerfile)  
[Alibaba Cloud PostgreSQL SSL Certificate Configuration](https://www.alibabacloud.com/blog/postgresql-ssl-certificate-configuration_599116)  
[Docker Hub Bitnami PostgreSQL with repmgr](https://hub.docker.com/r/bitnami/postgresql-repmgr)  

# Reload Configuration
If you change `repmgr.conf`
```sh
docker exec -it ${CONTAINER} kill -HUP $(docker exec -it ${CONTAINER} cat /tmp/repmgrd.pid)
```

---
---

# Failover
Postgres does not have built-in automatic failover, it requires 3rd party tooling to achieve. So If `pg-0` fails we use a utility called `pg_ctl` to promote the standby server to a primary server. After promoting the standby you could choose to setup a new standby server and configure replication to it or fail back to the original server once you have resolved the failure.

Let’s stop the primary server to simulate failure. This is executed on the Docker server hosting the primary PostgreSQL:
```sh
docker stop psql-01
```

Then log into postgres-2 and promote it to primary:
```sh
CONTAINER="psql-02"
# Check the cluster's status
docker exec ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf cluster show
# Confirm you can add an entry
docker exec psql -U postgres -h localhost --command "INSERT INTO customers (firstname, date_created) VALUES ('pg-1 failed', CURRENT_TIMESTAMP);"
```

Let’s restart the primary server. It should have become the primary?????
```sh
docker compose --file postgresql.yaml up --detach
```

# 
Use this command to check wheter a PostgreSQL server is the primary or the standby.
```sh
# CONTAINER="psql-01"
docker exec ${CONTAINER} psql -U postgres -h localhost --command  "select pg_is_in_recovery()" 2> /dev/null
```

Output if it's the `primary`. This node is the RW node.
```
 f
```

Output if it's the `standby`. This node is the RO node.
```
 t
```

> [!NOTE]  
> Note the space in front of the letter `t` or `f`.
