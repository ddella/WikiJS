# HAProxy on Docker

## Create HAProxy directory
Create the HAProxy directory on every Docker host. It will be mounted inside the container.
```sh
sudo install -v -d -g 99 -o 99 /opt/haproxy/etc/haproxy
```

> [!IMPORTANT]  
> Version 2.4+ of the container will run as `uid=99(haproxy) gid=99(haproxy) groups=99(haproxy)`. Make sure the directory have the correct owner and permission.

# Copying Certificates
The certificate and key for `HAProxy` should already have been created. The same certificate will be used on both servers. Don't forget to add the IP addresses of each HAProxy in your SAN if you like to access it via the IP. `HAProxy` expects the certificate and private key to be in one file.
```sh
SERVER="haproxy"
### Concatenate certificate and the private key
cat ${SERVER}.key ${SERVER}.crt > ${SERVER}.pem

### Copy the cert and sets the owner to 99 (haproxy inside the container)
sudo install -g 99 -o 99 -m 600 ${SERVER}.pem /opt/haproxy/etc/haproxy/${SERVER}.pem
### Copy the CA certificate and sets the owner to 99 (haproxy inside the container)
sudo install -g 99 -o 99 -m 600 wikijs-ca.crt /opt/haproxy/etc/haproxy/wikijs-ca.crt
### Copy the HTTPs certificate and sets the owner to 99 (haproxy inside the container)
sudo install -g 99 -o 99 -m 600 wikijs.pem /opt/haproxy/etc/haproxy/wikijs.pem
```

## Minimal configuration
```sh
PGSQL_PORT="5432"           # PostgreSQL listening TCP port (This is inside the container)
CHECK_PORT="5430"           # Check script listening TCP port (This is on the Docker host)
STR_PRIMARY_SQL="200"       # String returned if Primary PostgrSQL
STR_STANDBY_SQL="206"       # String returned if Standby PostgrSQL
RW_LISTENING_PORT="5435"    # TCP listening port on HAProxy for read/write to PostgreSQL server
RO_LISTENING_PORT="5436"    # TCP listening port on HAProxy for read only to PostgreSQL server
DOCKER_HOST_1="test1"       # Hostname of Docker host #1
DOCKER_HOST_2="test2"       # Hostname of Docker host #2
DNS_SERVERS="192.168.13.10:53"
DOMAIN="kloud.lan"

cat <<EOF | sudo tee /opt/haproxy/etc/haproxy/haproxy.cfg > /dev/null
global
  # global settings here

defaults
  # defaults here
  mode http                    # enable http mode which gives of layer 7 filtering
  timeout connect 5000ms       # max time to wait for a connection attempt to a server to succeed
  timeout client 50000ms       # max inactivity time on the client side
  timeout server 50000ms       # max inactivity time on the server side
  # timeout http-request    10s
  # timeout queue           1m
  # timeout connect         10s
  # timeout client          1m
  # timeout server          1m
  # timeout http-keep-alive 10s
  # timeout check           10s
  log global
  log stdout format raw local0 debug

resolvers nameservers
  # Your DNS server(s)
  nameserver ns1 ${DNS_SERVERS}

#---------------------------------------------------------------------
# FrontEnd Configuration for Wiki JS clients
#---------------------------------------------------------------------
frontend wikijs-frontend
  # Frontend that accepts requests from clients to Wiki JS
  # http-request redirect scheme https code 301 if !{ ssl_fc }
  mode http
  # Indicates the port on which incoming requests will be received.
  # bind *:9443 ssl crt /usr/local/etc/haproxy/wikijs.pem ca-file /usr/local/etc/haproxy/wikijs-ca.crt
  bind *:9443 ssl crt /usr/local/etc/haproxy/wikijs.pem
  default_backend wikijs-backend

#---------------------------------------------------------------------
# BackEnd Configuration for Wiki JS servers
#---------------------------------------------------------------------
backend wikijs-backend
  # Wiki JS servers that fulfill the requests
  mode http
  balance roundrobin
  # Had some issue at initial configuration, looks like cookie fixed it ???
  cookie SERVER insert indirect nocache
  server wikijs-01 wikijs-01.${DOMAIN}:3000 check cookie wikijs-01
  server wikijs-02 wikijs-02.${DOMAIN}:3000 check cookie wikijs-02

#---------------------------------------------------------------------
# FrontEnd Configuration for pgAdmin clients
#---------------------------------------------------------------------
frontend pgadmin-frontend
  # Frontend that accepts requests from clients to pgAdmin
  # http-request redirect scheme https code 301 if !{ ssl_fc }
  mode http
  # Indicates the port on which incoming requests will be received.
  bind *:9444 ssl crt /usr/local/etc/haproxy/wikijs.pem
  default_backend pgadmin-backend

#---------------------------------------------------------------------
# BackEnd Configuration for pgAdmin servers
#---------------------------------------------------------------------
backend pgadmin-backend
  # pgAdmin servers that fulfill the requests
  mode http
  balance roundrobin
  server pgadmin-01 pgadmin-01.${DOMAIN}:8080 check
  server pgadmin-02 pgadmin-02.${DOMAIN}:8080 check

listen stats  
  mode http
  bind *:7000 ssl crt /usr/local/etc/haproxy/wikijs.pem   # IP and port that 'stats' answers. 'crt' should contain both the certificate and the private key
  stats enable      # turns on stats module
  stats refresh 3s  # set auto-refresh rate
  stats uri /

listen ReadWrite
  bind *:${RW_LISTENING_PORT}
  mode tcp
  option tcp-check
  tcp-check send PING\r\n
  tcp-check expect string ${STR_PRIMARY_SQL}
  default-server inter 2s downinter 5s rise 3 fall 2 slowstart 60s maxconn 64 maxqueue 128 weight 100
  server psql-01 ${DOCKER_HOST_1}.${DOMAIN}:${PGSQL_PORT} check port ${CHECK_PORT} resolvers nameservers
  server psql-02 ${DOCKER_HOST_2}.${DOMAIN}:${PGSQL_PORT} check port ${CHECK_PORT} resolvers nameservers

listen ReadOnly
  bind *:${RO_LISTENING_PORT}
  mode tcp
  default-server inter 2s downinter 5s rise 3 fall 2 slowstart 60s maxconn 64 maxqueue 128 weight 100
  option tcp-check
  tcp-check send PING\r\n
  tcp-check expect string ${STR_STANDBY_SQL}
  server psql-01 ${DOCKER_HOST_1}.${DOMAIN}:${PGSQL_PORT} check port ${CHECK_PORT}
  server psql-02 ${DOCKER_HOST_2}.${DOMAIN}:${PGSQL_PORT} check port ${CHECK_PORT}
EOF
sudo chown 99:99 -R /opt/haproxy/etc/haproxy/haproxy.cfg
# If you change the config after the container is created, use the command below to reload it
# docker kill -s HUP haproxy-01
```

Configuration description:
- HAProxy is configured to use TCP mode.
- HAProxy service will start listening to ports 5000 and 5001.
  - TCP/5000 is for Read-Write connections
  - TCP/5001 is for Read-only connections
- Status check is done using the http-check feature on port 5430.
- All servers, pg-1, pg-2 and pg-31, are candidates for both Read-Write and Read-only connections.
- Based on the http-check and the status returned, it decides the current role.

## Test configuration
```sh
# Test the configuration file
docker run -it --rm --name test --network core-net -v /opt/haproxy/etc/haproxy:/usr/local/etc/haproxy:ro haproxy:3.1.0-alpine3.20 haproxy -V -f /usr/local/etc/haproxy/haproxy.cfg -c
# Print the version
docker run -it --rm --name test --network core-net -v /opt/haproxy/etc/haproxy:/usr/local/etc/haproxy:ro haproxy:3.1.0-alpine3.20 haproxy -v
```

Expected output:
```
Configuration file is valid
```

> [!TIP]
> I'm using a Docker custom network to have DNS resolution.

# Start the container
Don't use `docker run ...` but rather `docker compose ...`
```sh
# docker run -d --rm --name haproxy-01 --hostname haproxy-01 -v /opt/haproxy/etc/haproxy:/usr/local/etc/haproxy:ro --sysctl net.ipv4.ip_unprivileged_port_start=0 haproxy:3.1.0-alpine3.20
# docker run -it --rm --name haproxy-01 --hostname haproxy-01 haproxy:3.1.0-alpine3.20 /bin/sh
```

## Docker Compose
Since this is part of a bigger projet, just append this to the main docker compose file.

> [!NOTE]  
> Change the variable depending on the Docker host.  
> This will be appended to your existing Docker Compose `yaml` file.

```sh
# PRIMARY HAProxy
HOST="haproxy-01"
IPv4="192.168.255.101"

# STANDBY HAProxy
# HOST="haproxy-02"
# IPv4="192.168.255.102"

# Same for all HAProxy
YAML_FILE="docker.yaml"

cat <<EOF >>${YAML_FILE}

# Start the deployment
#   docker compose --file ${YAML_FILE} up --detach
#   docker compose --file ${YAML_FILE} restart
# Destroy the deployment
#   docker compose --file ${YAML_FILE} down
# Stop by keeps everything
#   docker compose --file ${YAML_FILE} stop
#   docker compose --file ${YAML_FILE} rm --force
# List the running container
#   docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Status}}\t{{.Names}}'
# Jump inside a container
#   docker exec -it ${HOST} /bin/sh
# Check logs
#   docker compose --file ${YAML_FILE} logs
# networks:
#   wikijs:
#     name: core-net
#     external: true
# services:
# ------------------------------------------------#
#        HAPROXY Container
# ------------------------------------------------#
  haproxy:
    image: haproxy:3.1.0-alpine3.20
    container_name: ${HOST}
    hostname: ${HOST}
    domainname: kloud.lan
    restart: unless-stopped
    # depends_on:
    #   - postgresql
    ports:
      #  HAProxy stats page
      - "9445:7000"
      # PostgreSQL Read Write
      - "5435:5435"
      # PostgreSQL Read Only
      - "5436:5436"
      # Wiki JS frontend
      - "9443:9443"
      # pgAdmin frontend
      - "9444:9444"
    volumes:
      - /opt/haproxy/etc/haproxy:/usr/local/etc/haproxy:ro
    networks:
      wikijs:
        # Define a static ip for the container. The containter can be accessible by others devices on the LAN network with this IP.
        ipv4_address: ${IPv4}
EOF
docker compose --file ${YAML_FILE} up --detach
```

# Reloading config
If you used a bind mount for the config and have edited your haproxy.cfg file, you can use HAProxy's graceful reload feature by sending a SIGHUP to the container:
```sh
docker kill -s HUP haproxy-01
```

# References
[HAProxy Documentation](https://docs.haproxy.org/)  
[Docker Hub HAProxy](https://hub.docker.com/_/haproxy/)  
[HAProxy configuration: The Four Essential Sections of an HAProxy Configuration](https://www.haproxy.com/blog/the-four-essential-sections-of-an-haproxy-configuration)  
