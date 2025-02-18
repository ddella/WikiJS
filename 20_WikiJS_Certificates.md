# Wiki.js TLS Certificates
This guide is about how to install Wiki.js with Docker Compose and an Nginx Reverse Proxy, all this on Ubuntu 24.04 LTS.

# TLS Certificates
For this projet you will need multiple certificates. I've decided to create a CA only for this projet and generate certificates signed by it.Certificates that will be needed:

| File               | Description                 |
|--------------------|-----------------------------|
| wikijs-ca.crt      | CA certificate for Wiki JS project |
| wikijs-ca.key      | CA private key for Wiki JS project |
| psql-crt.pem       | Postgres server certificate. Same for all PostgreSQL server |
| psql-key.pem       | Postgres server private key. Same for all PostgreSQL server |
| wikijs.crt         | Wiki JS server certificate. This certificate is for HAProxy |
| wikijs.key         | Wiki JS server private key. This certificate is for HAProxy  |

# Generate Private CA
This script will generate a CA root certificate with its private key. This CA will be used to sign the certificates for all the servers needed for WikiJS. You can use your own CA if you already have one.

> [!IMPORTANT]  
> Generate only one Root CA. The certificate and key needs to be copied to all your Docker hosts.

```sh
DOMAIN="kloud.lan"
CA_CERT="wikijs-ca"
SAN="DNS:localhost,DNS:*.localhost,DNS:${CA_CERT}.${DOMAIN},IP:127.0.0.1"

printf "\nMaking WikiJS Root CA...\n"
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:secp521r1 -pkeyopt ec_param_enc:named_curve -out ${CA_CERT}.key

openssl req -new -sha256 -x509 -key ${CA_CERT}.key -days 7305 \
-subj "/C=CA/ST=QC/L=Montreal/O=WikiJSRootCA/OU=${CA_CERT}/CN=wikijsca.${DOMAIN}" \
-addext "subjectAltName = ${SAN}" \
-addext "basicConstraints = critical,CA:TRUE" \
-addext "keyUsage = critical, digitalSignature, cRLSign, keyCertSign" \
-addext "subjectKeyIdentifier = hash" \
-addext "authorityKeyIdentifier = keyid:always, issuer" \
-addext "authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp" \
-addext "crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8080/crl/Intermediate-CA.crl" \
-out ${CA_CERT}.crt
```

This results in two files:
- The file `wikijs-ca.crt` is the CA certificate
- The file `wikijs-ca.key` is the CA private key

## Generate PostgreSQL Certificates
This following will generate the certificate and key for `PostgreSQL` nodes. The same certificate will be used for all PostgreSQL servers.
```sh
### Generate the CSR with an ECC private key
DOMAIN="kloud.lan"
SERVER="psql"
CA_CERT="wikijs-ca"
# SAN for all your PostgreSQL servers
SAN="DNS:localhost,DNS:*.localhost,IP:127.0.0.1,\
DNS:psql-01,\
DNS:psql-02,\
DNS:psql-01.$DOMAIN,\
DNS:psql-02.$DOMAIN,\
DNS:test1,\
DNS:test2,\
DNS:test1.$DOMAIN,\
DNS:test2.$DOMAIN,\
IP:192.168.13.31,\
IP:192.168.13.32,\
IP:192.168.255.31,\
IP:192.168.255.32"

printf "Generate the CSR with private key for [${SERVER}] ...\n"
openssl ecparam -name prime256v1 -genkey -out ${SERVER}.key

openssl req -new -sha256 -key ${SERVER}.key -subj "/C=CA/ST=QC/L=Montreal/O=${SERVER}/OU=IT/CN=postgres" \
-addext "subjectAltName = ${SAN}" \
-addext "basicConstraints = CA:FALSE" \
-addext "extendedKeyUsage = clientAuth,serverAuth" \
-addext "subjectKeyIdentifier = hash" \
-addext "keyUsage = nonRepudiation, digitalSignature, keyEncipherment, keyAgreement, dataEncipherment" \
-addext "authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp" \
-addext "crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8080/crl/Intermediate-CA.crl" \
-out ${SERVER}.csr

### Sign the CSR with the CA and keep all the extension from the CSR
printf "Signing the CSR with the CA (generates the certificate) for [${SERVER}] ...\n"
openssl x509 -req -sha256 -days 365 -in ${SERVER}.csr -CA ${CA_CERT}.crt -CAkey ${CA_CERT}.key -CAcreateserial -copy_extensions copyall -out ${SERVER}.crt &>/dev/null

### Delete the CSR
rm ${SERVER}.csr

### Copy the cert and sets the owner to 1001 (psql inside the container)
# sudo install -g 1001 -o 1001 -m 644 ${SERVER}.crt /opt/pg/postgresql/certs/{SERVER}.crt
# sudo install -g 1001 -o 1001 -m 600 ${SERVER}.key /opt/pg/postgresql/certs/{SERVER}.key
```

> [!IMPORTANT]  
> The `CN` inside the certificate needs to have the users `postgres` or you'll get the following error`psql: error: could not connect to server: FATAL:  connection requires a valid client certificate`.

This results in two files:
- The file `psql.crt` is the PostgreSQL certificate
- The file `psql.key` is the PostgreSQL private key

## Generate HAProxy Node Certificates
This following will generate the certificate and key for `HAProxy`. The same certificate will be used on both servers. I already have my Sub RootCA. Don't forget to add the IP addresses of each HAProxy in your SAN if you like to access it via the IP.
```sh
### Generate the CSR with an ECC private key
DOMAIN="kloud.lan"
SERVER="haproxy"
SAN="DNS:localhost,DNS:*.localhost,IP:127.0.0.1,\
DNS:${SERVER},\
DNS:${SERVER}.${DOMAIN},\
DNS:haproxy-01,\
DNS:haproxy-01.${DOMAIN},\
DNS:haproxy-02,\
DNS:test1,\
DNS:test2,\
DNS:test1.$DOMAIN,\
DNS:test2.$DOMAIN,\
IP:192.168.13.31,\
IP:192.168.13.32,\
DNS:haproxy-02.${DOMAIN}"

printf "Generate the CSR with private key for [${SERVER}] ...\n"
openssl ecparam -name prime256v1 -genkey -out ${SERVER}.key

openssl req -new -sha256 -key ${SERVER}.key -subj "/C=CA/ST=QC/L=Montreal/O=${SERVER}/OU=IT/CN=${SERVER}.${DOMAIN}" \
-addext "subjectAltName = DNS:localhost,DNS:*.localhost,DNS:${SERVER},DNS:${SERVER}.${DOMAIN},DNS:haproxy-01,DNS:haproxy-01.${DOMAIN},DNS:haproxy-02,DNS:haproxy-02.${DOMAIN},IP:127.0.0.1" \
-addext "basicConstraints = CA:FALSE" \
-addext "extendedKeyUsage = clientAuth,serverAuth" \
-addext "subjectKeyIdentifier = hash" \
-addext "keyUsage = nonRepudiation, digitalSignature, keyEncipherment, keyAgreement, dataEncipherment" \
-addext "authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp" \
-addext "crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8080/crl/Intermediate-CA.crl" \
-out ${SERVER}.csr

### Sign the CSR with the CA and keep all the extension from the CSR
printf "Signing the CSR with the CA (generates the certificate) for [${SERVER}] ...\n"
openssl x509 -req -sha256 -days 365 -in ${SERVER}.csr -CA ${CA_CERT}.crt -CAkey ${CA_CERT}.key -CAcreateserial -copy_extensions copyall -out ${SERVER}.crt &>/dev/null

### Delete the CSR
rm ${SERVER}.csr

### Concatenate certificate and the private key
cat ${SERVER}.key ${SERVER}.crt > ${SERVER}.pem

### Copy the cert and sets the owner to 99 (haproxy inside the container)
# sudo install -g 99 -o 99 -m 600 ${SERVER}.pem /opt/haproxy/etc/haproxy/${SERVER}.pem
```

This results in three files:
- The file `haproxy.crt` is the certificate (not neede anymore)
- The file `haproxy.key` is the private key (not neede anymore)
- The file `haproxy.pem` is the private key + the certificate, which is needed by HAProxy

> [!TIP]
> Copy the cetificates to all containers that might need it.

## Generate Wiki JS Node Certificates for HAProxy
This following will generate the certificate and key for `Wiki JS`. The certificate will be used on `HAProxy` for the frontend.

> [!IMPORTANT]  
> For this certificate I'm using my own intermediate CA.

```sh
### Generate the CSR with an ECC private key
DOMAIN="kloud.lan"
SERVER="wikijs"
CA_CERT="int"
SAN="DNS:localhost,DNS:*.localhost,IP:127.0.0.1,\
DNS:${SERVER},\
DNS:${SERVER}.${DOMAIN},\
IP:192.168.13.100, \
IP:192.168.13.31, \
IP:192.168.13.32"

printf "Generate the CSR with private key for [${SERVER}] ...\n"
openssl ecparam -name prime256v1 -genkey -out ${SERVER}.key

openssl req -new -sha256 -key ${SERVER}.key -subj "/C=CA/ST=QC/L=Montreal/O=${SERVER}/OU=IT/CN=${SERVER}.${DOMAIN}" \
-addext "subjectAltName = ${SAN}" \
-addext "basicConstraints = CA:FALSE" \
-addext "extendedKeyUsage = clientAuth,serverAuth" \
-addext "subjectKeyIdentifier = hash" \
-addext "keyUsage = nonRepudiation, digitalSignature, keyEncipherment, keyAgreement, dataEncipherment" \
-addext "authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp" \
-addext "crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8080/crl/Intermediate-CA.crl" \
-out ${SERVER}.csr

### Sign the CSR with the CA and keep all the extension from the CSR
printf "Signing the CSR with the CA (generates the certificate) for [${SERVER}] ...\n"
openssl x509 -req -sha256 -days 365 -in ${SERVER}.csr -CA ${CA_CERT}.crt -CAkey ${CA_CERT}.key -CAcreateserial -copy_extensions copyall -out ${SERVER}.crt &>/dev/null

### Delete the CSR
rm ${SERVER}.csr

### Concatenate certificate and the private key
cat ${SERVER}.key ${SERVER}.crt > ${SERVER}.pem

### Copy the cert and sets the owner to 99 (haproxy inside the container)
# sudo install -g 99 -o 99 -m 600 ${SERVER}.pem /opt/haproxy/etc/haproxy/${SERVER}.pem
```

This results in three files:
- The file `wikijs.crt` is the certificate (not neede anymore)
- The file `wikijs.key` is the private key (not neede anymore)
- The file `wikijs.pem` is the private key + the certificate, which is needed by HAProxy

> [!TIP]
> Copy the cetificates to all containers that might need it.
