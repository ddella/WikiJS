# Wiki JS
A modern, lightweight and powerful Wiki app built on NodeJS.

| Hostname    | Description                   |
|-------------|-------------------------------|
| wikijs-01   | Wiki JS frontend (web server) |
| wikijs-02   | Wiki JS frontend (web server) |

# Create the database
For Wiki JS to start, the database **must** exist. Create it from any Docker host that has the `psql` utility.

> [!IMPORTANT]  
> Do this on the primary PostgreSQL. 

```sh
PGBIN="/usr/bin/psql"
PGSQL_HOST="wikijs.kloud.lan" # Use the VIP for HAProxy!
PGSQL_PORT="5435"
PGSQL_USERNAME="postgres"
CRT_DIR="/etc/ssl/certs"
KEY_DIR="/etc/ssl/private"
sudo "${PGBIN}" "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" -t -c "CREATE DATABASE wikijs OWNER postgres;"
# Check that the database has been created
sudo "${PGBIN}" "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" -t -c "\l"
```

# Start a Wiki JS instance
If you want to provide your own SSL certificate and private key, you must mount a config file. Follow this [link](https://github.com/Requarks/wiki/blob/main/config.sample.yml) for an example of a configuration file.
I mounted the certificate and private key from my local disk.

## Create Wiki JS directory
Create the Wiki JS directory on the Docker host. It will be mounted inside the container.
```sh
sudo install -v -d -g 1000 -o 1000 /opt/wikijs/config
sudo install -v -d -g 1000 -o 1000 /opt/wikijs/data
```

# Generate Node Certificates
The certificate and key for `HAProxy` should already have been created. The same certificate will be used on both servers. Don't forget to add the IP addresses of each HAProxy in your SAN if you like to access it via the IP. `HAProxy` expects the certificate and private key to be in one file.
```sh
SERVER="psql"
CA="wikijs-ca"
### Copy the cert/key and sets the owner to 1000 (Wiki JS inside the container)
sudo install -g 1000 -o 1000 -m $(stat -c '%a' ${CA}.crt) ${CA}.crt /opt/wikijs/config/${CA}.crt
sudo install -g 1000 -o 1000 -m $(stat -c '%a' ${SERVER}.crt) ${SERVER}.crt /opt/wikijs/config/${SERVER}.crt
sudo install -g 1000 -o 1000 -m $(stat -c '%a' ${SERVER}.key) ${SERVER}.key /opt/wikijs/config/${SERVER}.key
```

## Configuration file
This is my configuration file. Execute on both servers:
```sh
cat <<EOF > config.yml
#######################################################################
# Wiki.js - CONFIGURATION                                             #
#######################################################################
# Full documentation + examples:
# https://docs.requarks.io/install

# ---------------------------------------------------------------------
# Port the server should listen to
# ---------------------------------------------------------------------
port: 3000

# ---------------------------------------------------------------------
# Database
# ---------------------------------------------------------------------
# Supported Database Engines:
# - postgres = PostgreSQL 9.5 or later
# - mysql = MySQL 8.0 or later (5.7.8 partially supported, refer to docs)
# - mariadb = MariaDB 10.2.7 or later
# - mssql = MS SQL Server 2012 or later
# - sqlite = SQLite 3.9 or later
db:
  type: postgres
  # This should be the VIP for HAProxy
  host: wikijs.kloud.lan
  port: 5435
  user: postgres
  # pass: wikijsrocks
  db: wikijs
  ssl: true

  # -> Full list of accepted options: https://nodejs.org/api/tls.html#tls_tls_createsecurecontext_options
  sslOptions:
    auto: false
    rejectUnauthorized: false
    ca: /config/wikijs-ca.crt
    cert: /config/psql.crt
    key: /config/psql.key

#######################################################################
# ADVANCED OPTIONS                                                    #
#######################################################################
# Do not change unless you know what you are doing!

# ---------------------------------------------------------------------
# SSL/TLS Settings
# ---------------------------------------------------------------------
# Consider using a reverse proxy (e.g. nginx) if you require more
# advanced options than those provided below.

ssl:
  enabled: false
  # port: 3443

  # # Provider to use, possible values: custom, letsencrypt
  # provider: custom

  # # ++++++ For custom only ++++++
  # # Certificate format, either 'pem' or 'pfx':
  # format: pem
  # # Using PEM format:
  # key: path/to/key.pem
  # cert: path/to/cert.pem
  # # Using PFX format:
  # pfx: path/to/cert.pfx
  # # Passphrase when using encrypted PEM / PFX keys (default: null):
  # passphrase: null
  # # Diffie Hellman parameters, with key length being greater or equal
  # # to 1024 bits (default: null):
  # dhparam: null

  # ++++++ For letsencrypt only ++++++
  # domain: wiki.yourdomain.com
  # subscriberEmail: admin@example.com

# ---------------------------------------------------------------------
# Database Pool Options
# ---------------------------------------------------------------------
# Refer to https://github.com/vincit/tarn.js for all possible options

pool:
  # min: 2
  # max: 10

# ---------------------------------------------------------------------
# IP address the server should listen to
# ---------------------------------------------------------------------
# Leave 0.0.0.0 for all interfaces

bindIP: 0.0.0.0

# ---------------------------------------------------------------------
# Log Level
# ---------------------------------------------------------------------
# Possible values: error, warn, info (default), verbose, debug, silly

logLevel: info

# ---------------------------------------------------------------------
# Log Format
# ---------------------------------------------------------------------
# Output format for logging, possible values: default, json

logFormat: default

# ---------------------------------------------------------------------
# Offline Mode
# ---------------------------------------------------------------------
# If your server cannot access the internet. Set to true and manually
# download the offline files for sideloading.

offline: false

# ---------------------------------------------------------------------
# High-Availability
# ---------------------------------------------------------------------
# Set to true if you have multiple concurrent instances running off the
# same DB (e.g. Kubernetes pods / load balanced instances). Leave false
# otherwise. You MUST be using PostgreSQL to use this feature.

ha: false

# ---------------------------------------------------------------------
# Data Path
# ---------------------------------------------------------------------
# Writeable data path used for cache and temporary user uploads.
dataPath: /wiki/data

# ---------------------------------------------------------------------
# Body Parser Limit
# ---------------------------------------------------------------------
# Maximum size of API requests body that can be parsed. Does not affect
# file uploads.
bodyParserLimit: 5mb
EOF
sudo install -g 1000 -o 1000 -m 644 config.yml /opt/wikijs/config/config.yml
```

## Docker Compose - PRIMARY
Since this is part of a bigger projet, just append this to the main docker compose file.
```sh
# Wiki JS Node 1
# HOST="wikijs-01"
# IPv4="192.168.255.11"

# Wiki JS Node 2
HOST="wikijs-02"
IPv4="192.168.255.12"

# Same for all HAProxy
YAML_FILE="docker.yaml"
cat <<EOF >>${YAML_FILE}

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
#   docker exec -it wikijs-01 /bin/sh
# Check logs
#   docker compose --file ${YAML_FILE} logs
# networks:
#   wikijs:
#     name: core-net
#     external: true
# services:
# ------------------------------------------------#
#        Wiki JS Container
# ------------------------------------------------#
  wikijs:
    # image: requarks/wiki:canary-2.5.0-dev.383
    image: requarks/wiki:2.5
    # image: linuxserver/wikijs:version-v2.5.305
    container_name: ${HOST}
    hostname: ${HOST}
    domainname: kloud.lan
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Montreal
      # - CONFIG_FILE="Path to the config.yml file"   # Doesn't work!
    volumes:
      - /opt/wikijs/config/config.yml:/wiki/config.yml
      - /opt/wikijs/config:/config
      - /opt/wikijs/data:/wiki/data   # ???
      # - /opt/wikijs/config/config.yml:/wiki/config.yml
      # - /opt/wikijs/config:/config
      # - /opt/wikijs/data:/wiki/data
      # - /opt/wikijs/config:/config    # linuxserver/wikijs:version-v2.5.305
      # - /opt/wikijs/data:/data        # linuxserver/wikijs:version-v2.5.305
    # ports:
    #   - 3000:3000
    networks:
      wikijs:
        # Define a static ip for the container. The containter can be accessible by others devices on the LAN network with this IP.
        ipv4_address: ${IPv4}
EOF
docker compose --file ${YAML_FILE} up --detach
docker logs ${HOST}
```

Output:
```
Loading configuration from /wiki/config.yml... OK
2024-12-12T13:38:26.182Z [MASTER] info: =======================================
2024-12-12T13:38:26.183Z [MASTER] info: = Wiki.js 2.5.0-dev.383 ===============
2024-12-12T13:38:26.183Z [MASTER] info: =======================================
2024-12-12T13:38:26.183Z [MASTER] info: Initializing...
2024-12-12T13:38:26.487Z [MASTER] info: Using database driver pg for postgres [ OK ]
2024-12-12T13:38:26.489Z [MASTER] info: Connecting to database...
2024-12-12T13:38:26.540Z [MASTER] info: Database Connection Successful [ OK ]
2024-12-12T13:38:26.574Z [MASTER] warn: DB Configuration is empty or incomplete. Switching to Setup mode...
2024-12-12T13:38:26.574Z [MASTER] info: Starting setup wizard...
2024-12-12T13:38:26.679Z [MASTER] info: Starting HTTP server on port 3000...
2024-12-12T13:38:26.679Z [MASTER] info: HTTP Server on port: [ 3000 ]
2024-12-12T13:38:26.680Z [MASTER] info: HTTP Server: [ RUNNING ]
2024-12-12T13:38:26.681Z [MASTER] info: 🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻
2024-12-12T13:38:26.681Z [MASTER] info: 
2024-12-12T13:38:26.681Z [MASTER] info: Browse to http://YOUR-SERVER-IP:3000/ to complete setup!
2024-12-12T13:38:26.681Z [MASTER] info: 
2024-12-12T13:38:26.681Z [MASTER] info: 🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺
```

> [!WARNING]  
> Use this url for the first login: `https://wikijs.kloud.lan:9443/login`  
> You should hit the frontend HAProxy for Wiki JS

```
# --------------------------------------------------------------------------------
#       !!!! ***** Congrats 🎉🎉🎉🥳🥳🥳🍾🍾🍾 You have a Wiki JS in H.A. ***** !!!!!
# --------------------------------------------------------------------------------
```

---

# Reloading config
If you used a bind mount for the config and have edited your `config.yml` file, you can use Wiki JS's graceful reload feature by sending a SIGHUP to the container:
```sh
# Doesn't work
docker kill -s HUP wikijs-01
docker restart wikijs-01
docker exec -it wikijs-01 /bin/bash
```

# Check logs
Check the logs for any errors. if everything is good, you should see a line with: `Browse to http://YOUR-SERVER-IP:3000/ to complete setup!`:
```sh
docker logs wikijs-01 -f
```

# Test
After the initial setup, you can use `https:
```sh
https://127.0.0.1:8443
```

# Troubleshooting
Jump inside the `Wiki JS` container:
```sh
docker exec -it  wikijs /bin/bash
```

Check the logs
```sh
docker logs -f wikijs
```

Start another container, with the same image, and verify that Postgres as started:
```sh
docker run -it --rm --network private --hostname wikijs-cli --name wikijs-cli \
mypostgres:16.0-alpine3.18 psql --host=wikijs-db --dbname=wiki --username=wikijs
```

# Backup Wiki JS database
The command below is used to back up the Wiki JS database running on a Docker container:
```sh
docker exec -t wikijs-db pg_dumpall --database=wiki --username=wikijs > wiki-dump.sql
```

- Container name: `wikijs-db`
- Database name: `wiki`
- Username: `wikijs`

The command `pg_dumpall` is a standard PostgreSQL tool for backing up s database.

# Restore Wiki JS database
The following command is used to restore the database:
```sh
cat wiki-dump.sql | docker exec -i wikijs-db psql --username=wikijs
```

# Cleanup
If you want to stop both containers:
```sh
docker rm -f wikijs
docker rm -f wikijs-db
```

# References
[Official Docker Postgres](https://hub.docker.com/_/postgres/)  
[Postgres TLS](https://www.postgresql.org/docs/current/ssl-tcp.html)  
[Wiki JS on Docker](https://docs.requarks.io/install/docker)  
[Wiki JS sample configuration file](https://github.com/Requarks/wiki/blob/main/config.sample.yml)  


# TCPDUMP & Wireshark
I installed `tcpdump` on the Postgres container.

```sh
# jump inside the container
docker exec -it wikijs-db /bin/bash
# update and install tcpdump
apk update && apk add tcpdump
# capture packets and dump to local filsesystem
tcpdump -i eth0 -nvvv -s0 -w mycap.pcap
```

Copy the `pcap` file from the container to the local host:
```sh
# exit the container and copy the pcap file on the host
docker cp wikijs-db:/mycap.pcap .
```

## Tshark
To get rid of the error: `tshark: Couldn't run /usr/bin/dumpcap in child process: Operation not permitted`
```sh
# chgrp wireshark /usr/bin/dumpcap 
chmod 4710 /usr/bin/dumpcap
```

Start `tshark`
```sh
tshark -i eth0 -f 'tcp port 5432' -w mycap.pcap
```

Copy the `pcap` file from the container to the local host:
```sh
# exit the container and copy the pcap file on the host
docker cp wikijs-db:/mycap.pcap .
```

# References
[Using the Docker image](https://docs.requarks.io/install/docker)  
[Wiki JS sample config file](https://github.com/Requarks/wiki/blob/main/config.sample.yml)  
[Linuxserver Wiki JS on Docker Hub](https://hub.docker.com/r/linuxserver/wikijs)  
[Requarks Wiki JS on GitHub](https://github.com/Requarks/wiki/pkgs/container/wiki)  
