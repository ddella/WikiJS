# Installation of Wiki.js
This guide is about how to install Wiki.js with Docker Compose and an Nginx Reverse Proxy, all this on Ubuntu 24.04 LTS.

We will be installing Wiki.js with Docker Compose for numerous advantages.

|Role|FQDN|IPv4 Docker Host|IPv4 Container|Port|Description|
|----|----|:----:|----|----|----|
|Wiki JS (VIP)|wikijs.kloud.lan|192.168.13.100|N/A|N/A|Virtual IP to hit a healthy HAProxy container|
|Wiki JS (VIP)|wikijs.kloud.lan|-|N/A|HTTPs/9443|Wiki JS web server|
|Wiki JS (VIP)|wikijs.kloud.lan|-|N/A|HTTPs/9445|HAProxy stats web server|
|Wiki JS|wikijs-01.kloud.lan|192.168.13.31|192.168.255.11|HTTP/3000|WikiJS Web Server|
|Wiki JS|wikijs-02.kloud.lan|192.168.13.32|192.168.255.12|HTTP/3000|WikiJS Web Server|
|PostgreSQL Database|psql-01.kloud.lan|192.168.13.31|192.168.255.21|TCP/5234|Database socket|
|PostgreSQL Database|psql-01.kloud.lan|-|N/A|TCP/5430|Health Check socket|
|PostgreSQL Database|psql-02.kloud.lan|192.168.13.32|192.168.255.22|TCP/5234|Database socket|
|PostgreSQL Database|psql-02.kloud.lan|-|N/A|TCP/5430|Health Check socket|
|pgAdmin Database|pgadmin-01.kloud.lan|192.168.13.31|192.168.255.31|HTTP/8080|pgAdmin Web Server|
|pgAdmin Database|pgadmin-02.kloud.lan|192.168.13.32|192.168.255.32|HTTP/8080|pgAdmin Web Server|
|HAProxy Load Balancer|haproxy-01.kloud.lan|192.168.13.31|192.168.255.101|HTTPs/9443|Wiki JS web server|
|HAProxy Load Balancer|haproxy-01.kloud.lan|-|N/A|HTTPs/9445|HAProxy stats web server|
|HAProxy Load Balancer|haproxy-02.kloud.lan|192.168.13.32|192.168.255.102|HTTPs/9443|Wiki JS web server|
|HAProxy Load Balancer|haproxy-02.kloud.lan|-|N/A|HTTPs/9445|HAProxy stats web server|

## Directories
The following directories will be used
|Description|Docker Host|Docker Container|
|----|----|----|
|PostgreSQL|/opt/psql/...|---|
||/opt/psql/postgresql|/bitnami/postgresql|
||/opt/psql/postgresql/tmp|/bitnami/postgresql/tmp|
||/opt/psql/postgresql/logs|/opt/bitnami/postgresql/logs|
|Replication manager|/opt/repmgr/...|---|
||/opt/repmgr/data|/bitnami/postgresql/data|
||/opt/repmgr/conf|/opt/bitnami/repmgr/conf|
|PostgreSQL Admin|/opt/pgadmin|/var/lib/pgadmin|
|HA Proxy|/opt/haproxy|---|
|Wiki JS|/opt/wikijs|---|


# Install Docker
Install Docker CE on your Ubuntu 24.04 LTS.
```sh
sudo apt update && sudo apt -y upgrade
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose docker-compose-plugin
sudo systemctl start docker.service
sudo systemctl status docker.service
# Need to log off and back in
sudo usermod -aG docker ${USER}
# exec sudo -s -u ${USER}
exec sudo -E -u "$USER" "$SHELL"
docker info
```

# DNS Entries
Make sure you have the following DNS entries before you begin. If you don't have your DNS entries and if some containers are *down*, some other container like `HAProxy` won't start.
```
; ***************************************
; *************** Wiki JS ***************
; ***************************************
wikijs          IN A      192.168.13.100
wikijs-01       IN A      192.168.255.11
wikijs-02       IN A      192.168.255.12
psql-01         IN A      192.168.255.21
psql-02         IN A      192.168.255.22
psql-01-ext     IN A      192.168.13.31
psql-02-ext     IN A      192.168.13.32
pgadmin-01      IN A      192.168.255.31
pgadmin-02      IN A      192.168.255.32
haproxy-01      IN A      192.168.255.101
haproxy-02      IN A      192.168.255.102
```

## Create working directory
Create a working directory to hold all the files needed for this project.
```sh
mkdir wikijs && cd wikijs
```

# TLS Certificates
For this projet you will need multiple certificates. I've decided to create a CA only for this projet and generate certificates signed by it.Certificates that will be needed:

| File               | Description                 |
|--------------------|-----------------------------|
| wikijs-ca.crt      | CA certificate for Wiki JS project |
| wikijs-ca.key      | CA private key for Wiki JS project |
| psql-crt.pem       | Postgres server certificate. Same for all PostgreSQL server |
| psql-key.pem       | Postgres server private key. Same for all PostgreSQL server |
| wikijs.crt         | Wiki JS server certificate. This certificate is for HAProxy |
| wikijs.key         | Wiki JS server private key. This certificate is for HAProxy |

[See the file 20_WikiJS_Certificates](./20_WikiJS_Certificates.md)

# Docker
We will create three containers on each Docker host to run WikiJS in full H.A. mode.
|Role|Description|
|----|----|
|Wiki JS|Wiki JS frontend application|
|PostgreSQL|The database for Wiji JS|
|HAProxy|The load balancer for the frontend and the database|
|pgAdmin|pgAdmin is a management tool for PostgreSQL and derivative relational databases|

## Install Docker-CE
To get started we need to install Docker Engine on Ubuntu. Before you install Docker Engine for the first time on a new host machine, you need to set up the Docker apt repository. Afterward, you can install and update Docker from the repository.

1. Install the Docker repository.
```sh
# Add Docker's official GPG key:
sudo apt-get update
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

2. Install the Docker packages
```sh
sudo apt-get -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose docker-compose-plugin
sudo systemctl start docker.service
sudo systemctl status docker.service
# Need to log off and back in
sudo usermod -aG docker ${USER}
# exec sudo -s -u ${USER}
exec sudo -E -u "$USER" "$SHELL"
docker info
```

## Create Docker Network
Every container will run on every Docker host. A network that span multiple Docker hosts will be needed for a full communication. The only one that does this is the `overlay` network. It creates VxLAN between every hosts to strech the network. You need `swarn` for this. You don't need to use it but it needs to be initialize on the primary host.

On the primary Docker, init swarm and create the overlay network that will span multiple Docker servers.:
```sh
docker swarm init
docker network create --driver=overlay --subnet=192.168.255.0/24 --gateway=192.168.255.1 --ip-range=192.168.255.224/27 --attachable core-net
```

> [!CAUTION]
> Make sure you don't assign a static IP address to a container in the `192.168.255.224/27` range. You will have strange behavior.  
> `docker swarm init ...` and `docker network create ...` should be run on the primary Docker host **ONLY**.

On all the other Docker host(s), you just need join the swarm and the network will be created automatically on the first use.
```sh
docker swarm join --token SWMTKN-1-3dixwktmq7d3l3qxsht9opomtxmlyamlxqrtckg5vjrlzhllia-eojz8pbmhquvwmaoqffkjpiu8 192.168.13.41:2377
# docker network create --driver=overlay --subnet=192.168.255.0/24 --gateway=192.168.255.1 --attachable core-net
```

> [!NOTE]  
> You don't create the network on the worker node.  
> If you forgot the token, you can retreive it from the master with the command `docker swarm join-token worker`

# Create PostgreSQL containers
[See the PostgreSQL tutorial](./30_PostgreSQL_HA.md)

# Install PostgreSQL Client
[See the PostgreSQL tutorial](40_PostgreSQL_client.md)  

# Create HAProxy container
[See the HAProxy tutorial](./50-HAProxy.md)

# References

## PostgreSQL
https://github.com/docker-library/postgres/blob/0b87a9bbd23f56b1e9e863ecda5cc9e66416c4e0/17/alpine3.20/Dockerfile
https://hub.docker.com/_/postgres
https://github.com/docker-library/postgres
https://www.postgresql.org/docs/current/index.html

## pgAdmin
https://hub.docker.com/r/dpage/pgadmin4/
https://www.pgadmin.org/download/pgadmin-4-container/
https://www.pgadmin.org/

## WikiJS
[WikiJS Configuration file sample](https://github.com/Requarks/wiki/blob/main/config.sample.yml)  
[Docker Hub WikiJS](https://hub.docker.com/r/requarks/wiki)  
