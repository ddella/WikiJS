
CONTAINER="psql-01"
docker exec ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf cluster show

CONTAINER="psql-02"
docker exec ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf cluster show

YAML_FILE="wikijs.yaml"
docker compose --file ${YAML_FILE} up --detach

# https://dba.stackexchange.com/questions/259170/is-it-possible-that-the-old-primary-follow-the-new-secondary
systemctl stop postgresql-12.service
/usr/pgsql-12/bin/repmgr -f /etc/repmgr/12/repmgr.conf node rejoin --force-rewind --verbose -d 'host=psql02 user=repmgr dbname=repmgr connect_timeout=2' --dry-run

# https://www.pythian.com/blog/technical-track/split-brain-solution-using-pg_rewind-in-postgresql
docker exec ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf node rejoin -d 'host=psql-01 user=repmgr dbname=repmgr connect_timeout=2' --verbose --force-rewind --dry-run

# https://github.com/bitnami/containers/blob/main/bitnami/postgresql-repmgr/README.md
Env. var.

https://www.dbi-services.com/blog/fast-setup-of-a-two-node-repmgr-cluster-with-auto-failover/

# repmgr.conf.sample
https://github.com/EnterpriseDB/repmgr/blob/master/repmgr.conf.sample

-----------
```sh
cat <<'EOF' | sudo tee /opt/repmgr/conf/failover.sh > /dev/null
#!/bin/bash
# Add this line in the file 'repmgr.conf'
# failover_validation_command='/opt/pg/repmgr/conf/failover.sh'
#
PRIMARY_IP="psql-01"
STANDBY_IP="psql-02"
# Path is inside the container
REPMGR_CONFIG="/opt/bitnami/repmgr/conf/repmgr.conf"
# Path is inside the container
PGLOG="/bitnami/postgresql/data/repmgr.log"

# Path is inside the container
PGBIN="/opt/bitnami/postgresql/bin/psql"
PGSQL_HOST="localhost"
PGSQL_PORT="5432"
PGSQL_USERNAME="postgres"
# Path is inside the container
CRT_DIR="/bitnami/postgresql/certs"
# Path is inside the container
KEY_DIR="/bitnami/postgresql/certs"

function echodate() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')]"
}

# Function to stop PostgreSQL on primary server
function stop_primary_db() {
  echo "$(echodate) [FAILOVER] Stopping primary PostgreSQL database" >> "$PGLOG"
  repmgr -f "${REPMGR_CONFIG}" node service --action=stop
}

# We perform a simple query that should return only 2 results, either 'f' or 't' on the PRIMARY
VALUE=$("${PGBIN}" "sslmode=verify-full sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${PRIMARY_IP} \
user=${PGSQL_USERNAME} port=${PGSQL_PORT}" -t -c "select pg_is_in_recovery()" /dev/null 2>&1 | tr -d '[:space:]')
printf "%s host=%s - value=%s\n" "$(echodate)" "${PRIMARY_IP}" "${VALUE}" >> "$PGLOG"

# We perform a simple query that should return only 2 results, either 'f' or 't' on the STANDBY
VALUE=$("${PGBIN}" "sslmode=verify-full sslrootcert=${CRT_DIR}/wikijs-ca.crt sslcert=${CRT_DIR}/psql.crt sslkey=${KEY_DIR}/psql.key host=${STANDBY_IP} \
user=${PGSQL_USERNAME} port=${PGSQL_PORT}" -t -c "select pg_is_in_recovery()" /dev/null 2>&1 | tr -d '[:space:]')
printf "%s host=%s - value=%s\n" "$(echodate)" "${STANDBY_IP}" "${VALUE}" >> "$PGLOG"

exit 0
EOF
sudo chmod +x /opt/repmgr/conf/failover.sh
sudo chown 1001:0 /opt/repmgr/conf/failover.sh
```
-----------
```sh
cat <<EOF | sudo tee /opt/repmgr/conf/repmgr.conf > /dev/null
# A unique integer greater than zero which identifies the node. 
node_id=1
# An arbitrary (but unique) string; we recommend using the server's hostname. 
node_name='psql-01'
# Database connection information as a conninfo string. All servers in the cluster must be able to connect to the local node using this string. 
conninfo='user=repmgr password=repmgrpassword host=psql-01 dbname=repmgr port=5432 connect_timeout=5'
# The node's data directory. This is needed by repmgr when performing operations when the PostgreSQL instance is not running and there's no other way of determining the data directory.
data_directory='/bitnami/postgresql/data'

location='dc-01'
log_level='NOTICE'
monitor_interval_secs='2'
connection_check_type='query'
# node_rejoin_timeout=30
# standby_reconnect_timeout=30
log_status_interval=20
#------------------------------------------------------------------------------
# Failover and monitoring settings (repmgrd)
#------------------------------------------------------------------------------
failover=automatic
priority=100
reconnect_attempts=3
reconnect_interval=5
promote_command='PGPASSWORD=repmgrpassword repmgr standby promote -f "/opt/bitnami/repmgr/conf/repmgr.conf" --log-level DEBUG --verbose'
follow_command='PGPASSWORD=repmgrpassword repmgr standby follow -f "/opt/bitnami/repmgr/conf/repmgr.conf" --log-to-file --upstream-node-id=%n'
monitoring_history='no'
failover_validation_command='/opt/bitnami/repmgr/conf/failover.sh %n %e %s "%t" "%d"'
election_rerun_interval=10
degraded_monitoring_timeout='5'
# standby_disconnect_on_failover=true
# monitoring_history=yes
async_query_timeout='20'
primary_visibility_consensus=false
pg_ctl_options='-o "--config-file=\"/opt/bitnami/postgresql/conf/postgresql.conf\" --external_pid_file=\"/opt/bitnami/postgresql/tmp/postgresql.pid\" --hba_file=\"/opt/bitnami/postgresql/conf/pg_hba.conf\""'
pg_basebackup_options=''
event_notification_command='/opt/bitnami/repmgr/events/router.sh %n %e %s "%t" "%d"'
ssh_options='-o "StrictHostKeyChecking no" -v'
use_replication_slots='1'
pg_bindir='/opt/bitnami/postgresql/bin'
EOF
sudo chown 1001:0 -R /opt/repmgr/conf/repmgr.conf
CONTAINER="psql-01"
# docker exec -it ${CONTAINER} repmgr -h psql-01 -U repmgr -d repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf standby clone --copy-external-config-files --dry-run
docker exec -it ${CONTAINER} kill -HUP $(docker exec -it ${CONTAINER} cat /tmp/repmgrd.pid)
# docker exec -it ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf --log-level DEBUG --verbose --force primary register

# prints the locations for 'postgresql.conf' and 'pg_hba.conf' files using psql utility as the postgres user:
docker exec ${CONTAINER} psql -U postgres -h localhost --command 'show config_file'
docker exec ${CONTAINER} psql -U postgres -h localhost --command 'show hba_file'
```

---------
---------
```sh
cat <<EOF | sudo tee /opt/repmgr/conf/repmgr.conf > /dev/null
# A unique integer greater than zero which identifies the node. 
node_id=2
# An arbitrary (but unique) string; we recommend using the server's hostname. 
node_name='psql-02'
# Database connection information as a conninfo string. All servers in the cluster must be able to connect to the local node using this string. 
conninfo='user=repmgr password=repmgrpassword host=psql-02 dbname=repmgr port=5432 connect_timeout=5'
# The node's data directory. This is needed by repmgr when performing operations when the PostgreSQL instance is not running and there's no other way of determining the data directory.
data_directory='/bitnami/postgresql/data'

location='dc-02'
log_level='NOTICE'
monitor_interval_secs='2'
connection_check_type='query'
# node_rejoin_timeout=30
# standby_reconnect_timeout=30
log_status_interval=20
#------------------------------------------------------------------------------
# Failover and monitoring settings (repmgrd)
#------------------------------------------------------------------------------
failover=automatic
priority=100
reconnect_attempts=3
reconnect_interval=5
promote_command='PGPASSWORD=repmgrpassword repmgr standby promote -f "/opt/bitnami/repmgr/conf/repmgr.conf" --log-level DEBUG --verbose'
follow_command='PGPASSWORD=repmgrpassword repmgr standby follow -f "/opt/bitnami/repmgr/conf/repmgr.conf" --log-to-file --upstream-node-id=%n'
monitoring_history='no'
failover_validation_command='/opt/bitnami/repmgr/conf/failover.sh %n %e %s "%t" "%d"'
election_rerun_interval=10
degraded_monitoring_timeout='5'
# standby_disconnect_on_failover=true
# monitoring_history=yes
async_query_timeout='20'
primary_visibility_consensus=false
pg_ctl_options='-o "--config-file=\"/opt/bitnami/postgresql/conf/postgresql.conf\" --external_pid_file=\"/opt/bitnami/postgresql/tmp/postgresql.pid\" --hba_file=\"/opt/bitnami/postgresql/conf/pg_hba.conf\""'
pg_basebackup_options=''
event_notification_command='/opt/bitnami/repmgr/events/router.sh %n %e %s "%t" "%d"'
ssh_options='-o "StrictHostKeyChecking no" -v'
use_replication_slots='1'
pg_bindir='/opt/bitnami/postgresql/bin'
EOF
sudo chown 1001:0 -R /opt/repmgr/conf/repmgr.conf
CONTAINER="psql-02"
# docker exec -it ${CONTAINER} repmgr -h psql-01 -U repmgr -d repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf standby clone --copy-external-config-files --dry-run
sudo rm /opt/repmgr/conf/repmgr.conf.tmp
docker exec -it ${CONTAINER} kill -HUP $(docker exec -it ${CONTAINER} cat /tmp/repmgrd.pid)
# Register the primary server using repmgr. Specify the repmgr configuration file using the -f option.
docker exec -it ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf --log-level DEBUG --verbose --force standby register
```
---------
Run the following command to list the standby nodes that have contacted the primary:
Primary Node as postgres User in PostgreSQL Shell (using Docker exec here)
```sh
CONTAINER="psql-01"
docker exec ${CONTAINER} psql -U postgres -h localhost --command "SELECT * FROM pg_stat_replication;"
docker exec ${CONTAINER} psql -U postgres -h localhost --command "SELECT * FROM pg_stat_wal_receiver;"
```

Each active standby node should have its own entry. Scan for the following details in the output:
  `application_name` should contain the `node_name` of the standby.
  The `client_addr` should indicate the IP address of the standby node.
  The `state` should be `streaming`.
  The `sync_state` is `async`.

Although updates are being sent to the standby, it is still not registered with repmgr. Until it is registered, it is not able to take over from the primary.
```sh
CONTAINER="psql-02"
docker exec -it ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf --log-level DEBUG --verbose --force standby register
docker exec ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf cluster show
docker exec ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf daemon status
```


---------
```sh
cat <<EOF | sudo tee /opt/repmgr/conf/repmgr.conf > /dev/null
node_id=1
node_name='psql-01'
election_rerun_interval=10
# =============================================================================
# Required configuration items
# =============================================================================
conninfo='user=repmgr password=repmgrpassword host=psql-01 dbname=repmgr port=5432 connect_timeout=5'
data_directory='/bitnami/postgresql/data'

#------------------------------------------------------------------------------
# Replication settings
#------------------------------------------------------------------------------
use_replication_slots=yes

#------------------------------------------------------------------------------
# Logging settings
#------------------------------------------------------------------------------
log_level=INFO
log_facility=STDERR
log_file='/bitnami/postgresql/data/repmgrd.log'

#------------------------------------------------------------------------------
# Environment/command settings
#------------------------------------------------------------------------------
# pg_bindir='/postgres15/app/postgres/bin'

#------------------------------------------------------------------------------
# external command options
#------------------------------------------------------------------------------
pg_ctl_options='-s -l /dev/null'
ssh_options='-q -o ConnectTimeout=10'

#------------------------------------------------------------------------------
# Standby follow settings
#------------------------------------------------------------------------------
primary_follow_timeout=60

#------------------------------------------------------------------------------
# Failover and monitoring settings (repmgrd)
#------------------------------------------------------------------------------
failover=automatic
priority=100
reconnect_attempts=3
reconnect_interval=5
promote_command='PGPASSWORD=repmgrpassword repmgr standby promote -f "/opt/bitnami/repmgr/conf/repmgr.conf" --log-level DEBUG --verbose'
follow_command='PGPASSWORD=repmgrpassword repmgr standby follow -f "/opt/bitnami/repmgr/conf/repmgr.conf" --log-to-file --upstream-node-id=%n'
monitoring_history=true
failover_validation_command='/opt/bitnami/repmgr/conf/failover.sh'
election_rerun_interval=10
#degraded_monitoring_timeout=-1
EOF
sudo chown 1001:1001 -R /opt/repmgr/conf/repmgr.conf
CONTAINER="psql-01"
docker exec -it ${CONTAINER} kill -HUP $(docker exec -it ${CONTAINER} cat /tmp/repmgrd.pid)
```
---------
```sh
cat <<EOF | sudo tee /opt/repmgr/conf/repmgr.conf > /dev/null
node_id=2
node_name='psql-02'
election_rerun_interval=10
# =============================================================================
# Required configuration items
# =============================================================================
conninfo='user=repmgr password=repmgrpassword host=psql-02 dbname=repmgr port=5432 connect_timeout=5'
data_directory='/bitnami/postgresql/data'

#------------------------------------------------------------------------------
# Replication settings
#------------------------------------------------------------------------------
use_replication_slots=yes

#------------------------------------------------------------------------------
# Logging settings
#------------------------------------------------------------------------------
log_level=INFO
log_facility=STDERR
log_file='/bitnami/postgresql/data/repmgrd.log'

#------------------------------------------------------------------------------
# Environment/command settings
#------------------------------------------------------------------------------
# pg_bindir='/postgres15/app/postgres/bin'

#------------------------------------------------------------------------------
# external command options
#------------------------------------------------------------------------------
pg_ctl_options='-s -l /dev/null'
ssh_options='-q -o ConnectTimeout=10'

#------------------------------------------------------------------------------
# Standby follow settings
#------------------------------------------------------------------------------
primary_follow_timeout=60

#------------------------------------------------------------------------------
# Failover and monitoring settings (repmgrd)
#------------------------------------------------------------------------------
failover=automatic
priority=100
reconnect_attempts=3
reconnect_interval=5
promote_command='PGPASSWORD=repmgrpassword repmgr standby promote -f "/opt/bitnami/repmgr/conf/repmgr.conf" --log-level DEBUG --verbose'
follow_command='PGPASSWORD=repmgrpassword repmgr standby follow -f "/opt/bitnami/repmgr/conf/repmgr.conf" --log-to-file --upstream-node-id=%n'
monitoring_history=true
failover_validation_command='/opt/bitnami/repmgr/conf/failover.sh'
election_rerun_interval=10
#degraded_monitoring_timeout=-1
EOF
sudo chown 1001:1001 -R /opt/repmgr/conf/repmgr.conf
CONTAINER="psql-02"
docker exec -it ${CONTAINER} kill -HUP $(docker exec -it ${CONTAINER} cat /tmp/repmgrd.pid)
```sh
---------
# repmgr directories
/opt/repmgr/conf/repmgr.conf:/opt/bitnami/repmgr/conf/repmgr.conf
/opt/pg/repmgr/data:/bitnami/postgresql/data

---------
# When we have a split brain, execute this command on the node that should be standby
CONTAINER="psql-02"
docker exec -it ${CONTAINER} repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf --log-level DEBUG --verbose node service --action=stop
---------
