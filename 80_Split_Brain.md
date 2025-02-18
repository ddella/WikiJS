# Split Brain
A split brain scenario is a rare case, where more than one PostgreSQL servers think its the PRIMARY. The split brain appears when the PRIMARY server is shutdown and powered on. In a Docker Swarm this cause a network partition where the primary node can't be reached by the standby nodes. As a result, one of the standby nodes will be promoted as a PRIMARY even though the real PRIMARY node exists.


- `psql-01` is the PRIMARY and active
- `psql-01` is shutdown
- `psql-02` becomes the PRIMARY
- `psql-01` is powered on
- both are PRIMARY

The command `docker stop <PRIMARY>` doesn't cause a split brain.

> [!IMPORTANT]  
> To be configured on STANDBY nodes **ONLY**

## Script
Execute this command to verify if there's a split brain, two or more PostgreSQL that are PRIMARY.

This should be configured on all the node(s) that should be standby in normal situation.

> [!NOTE]  
> Do not configure this script on the PRIMARY.

### Email
Script to send an email when a split brain is detected:
```sh
cat <<'EOF' | sudo tee /usr/local/bin/sb_email.sh > /dev/null
#!/bin/bash
# Send email when PostgreSQL split brain detected

TIME=$(date)
ID=$(tr -dc A-Za-z0-9 </dev/urandom | head -c 16)
MESSAGE_ID="<$ID@docker1.isociel.com>"

curl \
--url 'smtp.bellnet.ca' \
--mail-from daniel@isociel.com \
--mail-rcpt daniel@isociel.com \
--upload-file - <<EOB
From: Daniel <daniel@isociel.com>
To: Daniel DN <daniel@isociel.com>
Subject: Bell Fibe External IP
Date: $TIME
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: quoted-printable
Message-ID: $MESSAGE_ID

PostgreSQL Split Brain detected @ $TIME
=======================================
EOB
EOF
sudo chmod +x /usr/local/bin/sb_email.sh
```

### Split Brain script
```sh
cat <<'EOF' | sudo tee /usr/local/bin/splitbrain.sh > /dev/null
#!/bin/bash
# This script checks all PostgreSQL servers for their status (PRIMARY - STANDBY).
# If the script finds more than one PostgreSQL that are PRIMARY, it will demote all execpt one PRIMARY

PGBIN="/usr/bin/psql"
PSQL_01="192.168.13.31"
PSQL_02="192.168.13.32"
PGSQL_PORT="5432"
PGSQL_USERNAME="postgres"
CRT_DIR="/etc/ssl/certs"
KEY_DIR="/etc/ssl/private"
CONTAINER="psql-02"

# We perform a simple query that should return only 2 results:
#   'f' - Primary
#   't' - Standby
VALUE_PSQL_01=$(sudo "${PGBIN}" "sslmode=verify-full sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PSQL_01} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" -t -c "select pg_is_in_recovery()" /dev/null 2>&1 | tr -d '[:space:]')

VALUE_PSQL_02=$(sudo "${PGBIN}" "sslmode=verify-full sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PSQL_02} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" -t -c "select pg_is_in_recovery()" /dev/null 2>&1 | tr -d '[:space:]')

# Don't use '-it' from your Docker cli, the input device is not a TTY
if [ "${VALUE_PSQL_01}" = "${VALUE_PSQL_02}" ]; then
  # Demote a primary to standby
  printf "Split brain detected - %s will be demoted to STANDBY\n" "${PSQL_02}"
  docker exec ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf node service --action=stop
  /usr/local/bin/sb_email.sh > /dev/null 2>&1
else
  printf "Both PostgreSQL are healthy: psql-01=[%s] - psql-02=[%s]\n" "${VALUE_PSQL_01}" "${VALUE_PSQL_02}"
fi
exit 0
EOF
# Make it executable
sudo chmod +x /usr/local/bin/splitbrain.sh
```

> [!NOTE]  
> The script checks if two nodes are PRIMARY. In all other situation, the script consider the servers healthy, even if one of the server is down. Errors like the following are consider healthy is the sense that there's no split brain.  
> Connectionrefused  
> Noroutetohost

## Service
You need to create two files: one for service, other for timer with same name.

Service file `/etc/systemd/system/splitbrain.service`
```sh
cat <<EOF | sudo tee /etc/systemd/system/splitbrain.service > /dev/null
[Unit]
Description=Check for PostgreSQL split brain

[Service]
Type=oneshot
ExecStart=/usr/local/bin/splitbrain.sh

[Install]
WantedBy=multi-user.target
EOF
```

## Timer
Timer file `/etc/systemd/system/splitbrain.timer`

```sh
cat <<EOF | sudo tee /etc/systemd/system/splitbrain.timer > /dev/null
[Unit]
Description=Check for PostgreSQL split brain (Timer)

[Timer]
OnUnitActiveSec=30s
OnBootSec=30s

[Install]
WantedBy=timers.target
EOF
```

## Verification
Verify that the files you created above contain no errors:
```sh
sudo systemd-analyze verify /etc/systemd/system/splitbrain.*
```

> [!TIP]
> If the command returns no output, the files have passed the verification successfully.

## Enable Timer
Enable the timer to make sure that it is activated on boot and start the timer:
```sh
sudo systemctl daemon-reload
sudo systemctl enable --now splitbrain.timer
sudo systemctl status splitbrain.timer
```

# Logs
Check the logs for the new timer service.
```sh
journalctl -u splitbrain -f | grep splitbrain.sh
```

```sh
sudo systemctl list-timers --all
```

---

#
```sh
docker exec ${CONTAINER} psql -U postgres -h 192.168.13.100 -p 5436 --command "select * from customers;"
NAME=$(date '+%Y-%m-%d %H:%M:%S')
# Use the RW port
docker exec ${CONTAINER} psql -U postgres -h 192.168.13.100 -p 5435 --command "INSERT INTO customers (firstname, date_created) VALUES ('${NAME}', CURRENT_TIMESTAMP);"
# Use the RO port
docker exec ${CONTAINER} psql -U postgres -h 192.168.13.100 -p 5436 --command "select * from customers;"
```

```sh
sudo ${PGBIN} "sslmode=verify-ca sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PGSQL_HOST} user=${PGSQL_USERNAME} port=${PGSQL_PORT}" \
-t -c "select * from customers;"					
```

```sh
sudo systemctl cat splitbrain.timer
sudo systemctl cat splitbrain.service
sudo systemctl status splitbrain.timer
```
