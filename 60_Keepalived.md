# Keepalived
Install `keepalived` on a **Docker host** to check if the `HAProxy` container is alive.

# Install Keepalived (MASTER and BACKUP)
Let's get our hands dirty and learn about the installation and basic configuration of `keepalived` for our HAProxy. This section applies to both server **MASTER** and **BACKUP**.

This is a short guide on how to install `keepalived` package on Ubuntu 24.04:
```sh
PACKAGE="keepalived_2.3.2-1_amd64.deb"
curl -LO http://ftp.us.debian.org/debian/pool/main/k/keepalived/${PACKAGE}
sudo chown root:root ${PACKAGE}
sudo apt --fix-broken -y install ./${PACKAGE}
sudo rm -f ${PACKAGE}
```

# Check script
Create the script that checks if HAProxy is running. It's the same script for all Docker host:
```sh
cat <<'EOF' | sudo tee /etc/keepalived/check_haproxy.sh > /dev/null
#!/bin/sh
RESULT=$(curl -s -o /dev/null --insecure -w "%{http_code}" https://localhost:9445)
if [ "${RESULT}" != "200" ]; then
  exit 1
fi
EOF
sudo chmod 755 /etc/keepalived/check_haproxy.sh
```

> [!NOTE]  
> If you're using `HTTPs` make sure you have the option `--insecure` for the self-signed certificate.

# Configure `keepalived` on the PRIMARY
Create a Keepalived configuration file named `/etc/keepalived/keepalived.conf` on your **MASTER** node.

> [!NOTE]  
> Make sure you have all you DNS records for the cluster. I'm using it to set the `VIP` in `keepalived` configuration file.

```sh
INTERFACE=eth0
VIP=$(dig +short +search wikijs.kloud.lan | tail -1)
SUBNET_MASK=24
SECRET="WiKiJs!"
ROUTER_ID="10"

cat <<EOF | sudo tee /etc/keepalived/keepalived.conf > /dev/null
global_defs {
   notification_email {
     daniel@isociel.com
   }
   notification_email_from daniel@isociel.com
   smtp_server smtp.bellnet.ca
   smtp_connect_timeout 30
  # optimization option for advanced use
  max_auto_priority
  enable_script_security
  script_user root  
}

# Script to check whether HAPROXY is running or not
vrrp_script check_haproxy {
  script "/etc/keepalived/check_haproxy.sh"
  interval 3    # Check every 3 seconds
}

vrrp_instance HAPROXY {
  state MASTER
  interface ${INTERFACE}
  virtual_router_id ${ROUTER_ID}
  priority 255
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass ${SECRET}
  }
  virtual_ipaddress {
    ${VIP}/${SUBNET_MASK}
  }
  # Use the script above to check if we should fail over
  track_script {
    check_haproxy
  }
  smtp_alert
}
EOF
```

Restart `keepalived`:
```sh
sudo systemctl restart keepalived
sudo systemctl status keepalived
```

# Configure `keepalived` on your BACKUP
Create a Keepalived configuration file named `/etc/keepalived/keepalived.conf` on your **BACKUP** node.

> [!NOTE]  
> Make sure you have all you DNS records for the cluster. I'm using it to set the `VIP` in `keepalived` configuration file.

```sh
INTERFACE=eth0
VIP=$(dig +short +search wikijs.kloud.lan | tail -1)
SUBNET_MASK=24
SECRET="WiKiJs!"
ROUTER_ID="10"

cat <<EOF | sudo tee /etc/keepalived/keepalived.conf > /dev/null
global_defs {
   notification_email {
     daniel@isociel.com
   }
   notification_email_from daniel@isociel.com
   smtp_server smtp.bellnet.ca
   smtp_connect_timeout 30
  # optimization option for advanced use
  max_auto_priority
  enable_script_security
  script_user root  
}

# Script to check whether HAPROXY is running or not
vrrp_script check_haproxy {
  script "/etc/keepalived/check_haproxy.sh"
  interval 3    # Check every 3 seconds
}

vrrp_instance HAPROXY {
  state MASTER
  interface ${INTERFACE}
  virtual_router_id ${ROUTER_ID}
  priority 200
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass ${SECRET}
  }
  virtual_ipaddress {
    ${VIP}/${SUBNET_MASK}
  }
  # Use the script above to check if we should fail over
  track_script {
    check_haproxy
  }
  smtp_alert
}
EOF
```

Restart `keepalived`:
```sh
sudo systemctl restart keepalived
sudo systemctl status keepalived
```

## Parameters
`keepalived `Parameters:

- state MASTER/BACKUP: the state that the router will start in.
- interface ens3: interface VRRP protocol messages should flow.
- virtual_router_id: An ID that both servers should agreed on.
- priority: number for master/backup election – higher numerical means higher priority.
- advert_int: backup waits this long (multiplied by 3) after messages from master fail before becoming master
- authentication: a clear text password authentication, no more than 8 caracters.
- virtual_ipaddress: the virtual IP that the servers will share
- track_script: Check if `HAProxy` is running, if not `keepalived` fail to the backup

# Logs
Check the system logs for any errors:
```sh
sudo journalctl -u keepalived
```

# TCPDUMP
Check for VRRP traffic using tools like tcpdump:
```sh
sudo tcpdump -i eth0 vrrp
```

> [!TIP]
> Replace `eth0` with your network interface name.
