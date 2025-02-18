# PostgreSQL client
We need to install `psql` client on every **Docker host**. This is required to check which PostgreSQL server is the PRIMARY.

To manually configure the apt repository and install `psql` client on **every Docker host**, follow these steps:
```sh
# Import the repository signing key
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
# Create the repository configuration file
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo apt update
# We only need the client and version 17
sudo apt install -y postgresql-client-17
```

The repository contains many different packages including third party addons. The most common and important packages are (substitute the version number as required):

|Package|Description|
|----|----|
|postgresql-client-17|client libraries and client binaries|
|postgresql-17|core database server|
|postgresql-doc-17|documentation|
|libpq-dev|libraries and headers for C language frontend development|
|postgresql-server-dev-17|libraries and headers for C language backend development|

## Get prompt
Check that the client works with TLS certificate. You should get the `psql` prompt. Type "exit" to quit. You need to be in the same directory where the PostgreSQL certificates are.
```sh
psql "sslmode=verify-ca sslrootcert=wikijs-ca.crt sslcert=psql.crt sslkey=psql.key host=localhost port=5432 user=postgres"
```

Output:
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: postgresql)
Type "help" for help.

postgres=# exit
```

## Execute a command
You can execute a command with the `psql` client utility. I'm using TLS, so you will need the client certificate, private key and Root CA certificate.
```sh
# 'f' = Read-Write nodes
# 't' = Read-Only nodes
psql "sslmode=verify-full sslrootcert=wikijs-ca.crt sslcert=psql.crt sslkey=psql.key host=localhost port=5432 user=postgres" -t -c "select pg_is_in_recovery()" | tr -d '[:space:]'
# Get data from a table we created earlier to test the database
psql "sslmode=verify-full sslrootcert=wikijs-ca.crt sslcert=psql.crt sslkey=psql.key host=localhost port=5432 user=postgres" -t -c "select * from customers;"
```

## Script for HAProxy
This script, when executed, will return a responde code. This will be used by a load balancer to 
- Check if the local PostgreSQL server is running
- and confirms if it's a primary or standby node

It needs to be on every Docker hosts.
```sh
cat <<'EOF' | sudo tee /usr/local/bin/pgcheck.sh
#!/bin/bash
# This script checks if a PostgreSQL is running on localhost and returns a value that indicates if it's the primary or standby.
#
# It will return:
#   "200" --- if PostgreSQL is primary
#   "206" --- if PostgreSQL is standby
#   "503" --- if PostgreSQL is not running
# The purpose of this script is to make 'HAproxy' capable of monitoring PostgreSQL properly and knowing which is the role of the server:  either 'primary' or 'standby'.

PGBIN="/usr/bin/psql"
PGSQL_HOST="localhost"
PGSQL_PORT="5432"
PGSQL_USERNAME="postgres"
CRT_DIR="/etc/ssl/certs"
KEY_DIR="/etc/ssl/private"

# We perform a simple query that should return only 2 results, either 'f' or 't'
VALUE=$(sudo "${PGBIN}" "sslmode=verify-full sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" -t -c "select pg_is_in_recovery()" /dev/null 2>&1 | tr -d '[:space:]')
H=$(hostname -s)
if [ "${VALUE}" == "t" ]; then
  printf "206\r\n"
elif [ "${VALUE}" == "f" ]; then
  printf "200\r\n"
else
  printf "503\r\n"
fi
EOF
```

Copy the files in their appropriate locations and make sure the script is executable.
```sh
sudo chmod +x /usr/local/bin/pgcheck.sh
sudo install -m $(stat -c '%a' psql.crt) wikijs-ca.crt /etc/ssl/certs/wikijs-ca.crt
sudo install -m $(stat -c '%a' psql.crt) psql.crt /etc/ssl/certs/psql.crt
sudo install -m $(stat -c '%a' psql.key) psql.key /etc/ssl/private/psql.key
# Here I'm running the script, you should get either 200 or 206
/usr/local/bin/pgcheck.sh
```

The last line execute the srcipt. You should see either `200` or `206` depending if your on a primary or standby PostgreSQL server.

# Systemd
This section will create a TCP daemon and a service that:
- Will listen on a specific port
- Will execute the `pgcheck.sh`
- Will return the value to the client

The script will be run locally on every Docker host and executed by a load balancer. Based on the return value, the load balancer will know which database is the primary (R.W.) or the standby (R.O.).
```sh
cat <<EOF | sudo tee /usr/lib/systemd/system/pgcheck@.service > /dev/null
[Unit]
Description=PostgreSQL check service for HAProxy
After=network-online.target

[Service]
# Note the '-' to make systemd ignore the exit code
ExecStart=-/bin/bash /usr/local/bin/pgcheck.sh
Type=simple
StandardInput=socket

[Install]
WantedBy=multi-user.target
EOF
```

```sh
cat <<EOF | sudo tee /usr/lib/systemd/system/pgcheck.socket > /dev/null
[Unit]
Description=Socket activation for HAProxy to check PostgreSQL Server with psql

[Socket]
ListenStream=5430
Accept=yes

[Install]
WantedBy=sockets.target
EOF
```

## Start Listening Socket
```sh
sudo systemctl daemon-reload
sudo systemctl enable --now pgcheck.socket
sudo systemctl status pgcheck.socket
```

Output:
```
Created symlink /etc/systemd/system/sockets.target.wants/pgcheck.socket → /usr/lib/systemd/system/pgcheck.socket.
```

## Test it
Test the listening service. Just open a TCP connection and send a `\r\n` to close it:
```sh
# On Linux
echo "" | netcat 127.0.0.1 5430
# On macOS
# echo "" | nc 192.168.13.31 5430
# echo "" | nc 192.168.13.32 5430
```

The last line execute the srcipt. You should see either `200` or `206` depending if your on a primary or standby PostgreSQL server.

Watch the log for the echo:
```sh
journalctl -f --user-unit pgcheck.service
```

# Remove Service (Don't do this 😉)
In case you want to remove it from your Docker host.
```sh
sudo systemctl stop pgcheck.socket
sudo systemctl disable pgcheck.socket
sudo rm /usr/lib/systemd/system/pgcheck.socket
sudo rm /usr/lib/systemd/system/pgcheck@.service
sudo systemctl daemon-reload
```

# References
[Linux downloads (Ubuntu) Ubuntu](https://www.postgresql.org/download/linux/ubuntu/)  
