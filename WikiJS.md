# Wiki JS
A modern, lightweight and powerful Wiki app built on NodeJS.

| Hostname    | Description                   |
|-------------|-------------------------------|
| wikijs      | Wiki JS frontend (web server) |
| wikijs-db   | Postgres database server      |

# Start a Postgres instance
I created a directory `postgres` to build the Docker image. Feel free to do what's best for you.

# Generate TLS certificates
This is a simple and dirty script to generate the certificates needed for this project. I used OpenSSL 3.1.x and all keys are ECC.
```sh
cat <<EOF > ecc_gen_chain.sh
#!/bin/sh
#
# This script generates certificates for a fake RootCA, intermediate CA, server and a client using ECC keys.
# The CA and SubCA have 521-bit private keys. The server and client have 256-bit private keys.
# All the files are in '.pem' format. The private keys are NOT encrypted and are in PKCS#8 format.
#
# The following files are generated in the current directory:
#   ca-key.pem          Root CA private key
#   ca-crt.pem          Root CA certificate
#   ca-chain.pem        Root CA and SubRoot CA certificates in one '.pem' file
#   int-key.pem         SubRoot CA private key
#   int-csr.pem         SubRoot CA certificate signing request
#   int-crt.pem         SubRoot CA certificate
#   wikijs-key.pem      Wikijs private key
#   wikijs-csr.pem      Wikijs certificate signing request
#   wikijs-crt.pem      Wikijs certificate
#   wikijs-db-key.pem   Postgres private key
#   wikijs-db-csr.pem   Postgres certificate signing request
#   wikijs-db-crt.pem   Postgres certificate
#
# HOWTO use it: ./ecc_gen_chain.sh
#
# Works with 'bash' and 'zsh' shell on macOS and Linux. Make sure you have OpenSSL *** in your PATH ***.
#
# *WARNING*
# This script was made for educational purposes ONLY.
# USE AT YOUR OWN RISK!"

# printf "Deleting old certificates\n"
# rm -f *.pem
# rm -f *.srl

printf "\nMaking Root CA...\n"
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:secp521r1 -pkeyopt ec_param_enc:named_curve -out ca-key.pem
openssl req -new -sha256 -x509 -key ca-key.pem -days 7300 \
-subj "/C=CA/ST=QC/L=Montreal/O=RootCA/OU=IT/CN=RootCA.com" \
-addext "subjectAltName = DNS:localhost,DNS:*.localhost,DNS:RootCA.com,IP:127.0.0.1" \
-addext "basicConstraints = critical,CA:TRUE" \
-addext "keyUsage = critical, digitalSignature, cRLSign, keyCertSign" \
-addext "subjectKeyIdentifier = hash" \
-addext "authorityKeyIdentifier = keyid:always, issuer" \
-addext "authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp" \
-addext "crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8080/crl/Intermediate-CA.crl" \
-out ca-crt.pem

printf "\nMaking Intermediate CA With Trust...\n"
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:secp521r1 -pkeyopt ec_param_enc:named_curve -out int-key.pem
openssl req -new -sha256 -key int-key.pem \
-subj "/C=CA/ST=QC/L=Montreal/O=IntermediateCA/OU=IT/CN=SubRootCA.com" \
-addext "subjectAltName = DNS:localhost,DNS:*.localhost,DNS:SubRootCA.com,IP:127.0.0.1" \
-addext "basicConstraints = critical, CA:TRUE, pathlen:0" \
-addext "keyUsage = critical, digitalSignature, cRLSign, keyCertSign" \
-addext "subjectKeyIdentifier = hash" \
-addext "authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp" \
-addext "crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8080/crl/Intermediate-CA.crl" \
-out int-csr.pem
openssl x509 -req -sha256 -days 365 -in int-csr.pem -CA ca-crt.pem -CAkey ca-key.pem -CAcreateserial \
-extfile - <<<"subjectAltName = DNS:localhost,DNS:*.localhost,DNS:SubRootCA.com,IP:127.0.0.1
basicConstraints = critical, CA:TRUE, pathlen:0
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always, issuer
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp
crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8000/crl/Intermediate-CA.crl" \
-out int-crt.pem

printf "\nMaking Wiki JS Public/Private Key and Certificate...\n"
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:prime256v1 -pkeyopt ec_param_enc:named_curve -out wikijs-key.pem
openssl req -new -sha256 -key wikijs-key.pem -subj "/C=CA/ST=QC/L=Montreal/O=WikiJS/OU=IT/CN=wikijs" \
-addext "subjectAltName = DNS:localhost,DNS:*.localhost,DNS:wikijs,IP:127.0.0.1" \
-addext "basicConstraints = CA:FALSE" \
-addext "extendedKeyUsage = serverAuth,clientAuth" \
-addext "subjectKeyIdentifier = hash" \
-addext "keyUsage = nonRepudiation, digitalSignature, keyEncipherment, keyAgreement, dataEncipherment" \
-addext "authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp" \
-addext "crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8080/crl/Intermediate-CA.crl" \
-out wikijs-csr.pem
openssl x509 -req -sha256 -days 365 -in wikijs-csr.pem -CA int-crt.pem -CAkey int-key.pem -CAcreateserial \
-extfile - <<<"subjectAltName = DNS:localhost,DNS:*.localhost,DNS:wikijs,IP:127.0.0.1
basicConstraints = CA:FALSE
extendedKeyUsage = serverAuth,clientAuth
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid, issuer
keyUsage = nonRepudiation, digitalSignature, keyEncipherment, keyAgreement, dataEncipherment
authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp
crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8000/crl/Intermediate-CA.crl" \
-out wikijs-crt.pem

printf "\nMaking Postgres Public/Private Key and Certificate...\n"
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:prime256v1 -pkeyopt ec_param_enc:named_curve -out wikijs-db-key.pem
openssl req -new -sha256 -key wikijs-db-key.pem -subj "/C=CA/ST=QC/L=Montreal/O=Postgres/OU=IT/CN=wikijs-db" \
-addext "subjectAltName = DNS:localhost,DNS:*.localhost,DNS:wikijs-db,IP:127.0.0.1" \
-addext "basicConstraints = CA:FALSE" \
-addext "extendedKeyUsage = serverAuth,clientAuth" \
-addext "subjectKeyIdentifier = hash" \
-addext "keyUsage = nonRepudiation, digitalSignature, keyEncipherment, keyAgreement, dataEncipherment" \
-addext "authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp" \
-addext "crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8080/crl/Intermediate-CA.crl" \
-out wikijs-db-csr.pem
openssl x509 -req -sha256 -days 365 -in wikijs-db-csr.pem -CA int-crt.pem -CAkey int-key.pem -CAcreateserial \
-extfile - <<<"subjectAltName = DNS:localhost,DNS:*.localhost,DNS:wikijs-db,IP:127.0.0.1
basicConstraints = CA:FALSE
extendedKeyUsage = serverAuth,clientAuth
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid, issuer
keyUsage = nonRepudiation, digitalSignature, keyEncipherment, keyAgreement, dataEncipherment
authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp
crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8000/crl/Intermediate-CA.crl" \
-out wikijs-db-crt.pem

printf "\nMaking partial Chain of Root and SubRoot CA for Postgres certificate...\n"
cat ca-crt.pem int-crt.pem > ca-chain-crt.pem

printf "Verfying IntermediateCA via RootCA...\n"
openssl verify -CAfile ca-crt.pem int-crt.pem

printf "Verfying Partial Chain with Server cert via IntermediateCA...\n"
openssl verify -no-CAfile -no-CApath -partial_chain -trusted int-crt.pem wikijs-crt.pem

printf "Verfying Server cert via IntermediateCA via RootCA...\n"
openssl verify -CAfile ca-crt.pem -untrusted int-crt.pem wikijs-crt.pem

printf "Verfying Server via CA and SubCA Chain...\n"
openssl verify -CAfile ca-chain-crt.pem wikijs-crt.pem

printf "\nVerfying Partial Chain with Client cert via IntermediateCA...\n"
openssl verify -no-CAfile -no-CApath -partial_chain -trusted int-crt.pem wikijs-db-crt.pem

printf "Verfying Client cert via IntermediateCA via RootCA...\n"
openssl verify -CAfile ca-crt.pem -untrusted int-crt.pem wikijs-db-crt.pem

printf "Verfying Client via CA and SubCA Chain...\n"
openssl verify -CAfile ca-chain-crt.pem wikijs-db-crt.pem

printf "Removing Certificate Signing Requests...\n"
rm -f *-csr.pem
EOF
```

Generate all the certificates and keys:
```sh
chmod +x ecc_gen_chain.sh
./ecc_gen_chain.sh
```

## Configure TLS
Files that will be needed:

| File               | Description                 |
|--------------------|-----------------------------|
| ca-chain-crt.pem   | CA chain (includes SubCA)   |
| int-crt.pem        | Sub Certificate Authority   |
| wikijs-db-crt.pem  | Postgres server certificate |
| wikijs-db-key.pem  | Postgres server private key |
| wikijs-crt.pem     | Wiki JS server certificate  |
| wikijs-key.pem     | Wiki JS server private key  |
| wikijs-db-crt-chain.pem | Postgres server + SubCA certificates |

> [!NOTE]  
> Intermediate certificates that chain up to existing root certificates can also appear in the `ssl_ca_file` file if you wish to avoid storing them on clients (assuming the root and intermediate certificates were created with v3_ca extensions).
```sh
cat ca-crt.pem int-crt.pem > ca-chain-crt.pem
```

Create the file `config.sql` with settings that will be appended to `postgresql.auto.conf`. This will be the TLS configuration commands.
```sql
cat <<EOF > config.sql
ALTER SYSTEM SET ssl = on;
ALTER SYSTEM SET ssl_ca_file = '/etc/ssl/certs/ca-chain-crt.pem';
ALTER SYSTEM SET ssl_cert_file = '/etc/ssl/certs/wikijs-db-crt.pem';
ALTER SYSTEM SET ssl_key_file = '/etc/ssl/private/wikijs-db-key.pem';
EOF
```

## Build a new image
I built a new Postgres image with the certificates, private key and modification to be applied to the configuration file `${$PGDATA}/postgresql.auto.conf`

> [!IMPORTANT]  
> On Unix systems, the permissions on the private key must disallow any access to world or group. To achieve this, do:
> - Change the group of the private key file to `postgres`. The file is still owned by `root`.
> - Have group read access, with `0640` permissions

```Dockerfile
cat <<EOF > Dockerfile
# docker build -t mypostgres:16.0-alpine3.18 .
FROM postgres:16.0-alpine3.18
COPY --chmod=0755 config.sql /docker-entrypoint-initdb.d/.
COPY --chmod=0640 wikijs-db-key.pem /etc/ssl/private/.
RUN chown :postgres /etc/ssl/private/wikijs-db-key.pem
COPY wikijs-db-crt.pem /etc/ssl/certs/.
COPY ca-chain-crt.pem /etc/ssl/certs/.
EOF
```

> [!NOTE]  
> I used the image `postgres:16.0-alpine3.18` which is only `239MB`.

Build the image with the command:
```sh
docker image rm mypostgres:16.0-alpine3.18
docker build -t mypostgres:16.0-alpine3.18 .
```

## Start Postgres
I started Postgres in a custom network to get the DNS resolution.
```sh
docker run -d --rm --network private --hostname wikijs-db --name wikijs-db \
-v ./data:/var/lib/postgresql/data \
-e POSTGRES_PASSWORD=wikijsrocks \
-e POSTGRES_DB=wiki \
-e POSTGRES_USER=wikijs mypostgres:16.0-alpine3.18
```

Check the logs for any errors. if everything is good, the last line should read: `database system is ready to accept connections`:
```sh
docker logs wikijs-db
```

# Start a Wiki JS instance
If you want to provide your own SSL certificate and private key, you must mount a config file. Follow this [link](https://github.com/Requarks/wiki/blob/main/config.sample.yml) for an example of a configuration file.
I mounted the certificate and private key from my local disk.

## Configuration file
This is the modification of my configuration file that I used:
```yaml
cat <<EOF > wikijs-config.yaml
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
  host: wikijs-db
  port: 5432
  user: wikijs
  pass: wikijsrocks
  db: wiki
  ssl: true

  # -> Full list of accepted options: https://nodejs.org/api/tls.html#tls_tls_createsecurecontext_options
  sslOptions:
    auto: false
    rejectUnauthorized: false
    ca: ca-chain-crt.pem
    cert: wikijs-crt.pem
    key: wikijs-key.pem

# ---------------------------------------------------------------------
# SSL/TLS Settings
# ---------------------------------------------------------------------
# Consider using a reverse proxy (e.g. nginx) if you require more
# advanced options than those provided below.
ssl:
  enabled: true
  port: 3443
  provider: custom
  format: pem
  key: wikijs-key.pem
  cert: wikijs-crt.pem

EOF
```

## Start Wiki JS
After starting the server, use the `http` and finalize the configuration. `https` won't work for the initial configuration
```sh
docker run -d -p 8080:3000 -p 8443:3443 --name wikijs --hostname wikijs --network private \
-v ./wikijs-config.yaml:/wiki/config.yml \
-v ./ca-chain-crt.pem:/wiki/ca-chain-crt.pem \
-v ./wikijs-crt.pem:/wiki/wikijs-crt.pem \
-v ./wikijs-key.pem:/wiki/wikijs-key.pem \
ghcr.io/requarks/wiki:2.5
```

Check the logs for any errors. if everything is good, you should see a line with: `Browse to http://YOUR-SERVER-IP:3000/ to complete setup!`:
```sh
docker logs wikijs
```

> [!WARNING]  
> use the `http` for the first login. `http://127.0.0.1:8080`

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
