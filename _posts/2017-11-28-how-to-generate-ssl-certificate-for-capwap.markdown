---
title: "How to generate ssl certificate for capwap"
date: "2017-11-28 23:59:35 +0800"
toc: true
categories: [work]
tags: [ssl, pem, openssl]
---

## Revision

|Version|Author | Date | Comments|
|:-|:-|:-|:-|
|v0|Frank Zhou | Nov 28, 2017 | Draft|


## Step 1 Generate RSA private keys for CA/Cleint/Server

During generate private key with encryption, you will be asked twice for typing password.

1. Generate rsa_private key (with password CA_PASSWORD, 'prova' for example) for ca.
```
openssl genrsa -aes256 -out ca_rsa_private.key 2048
```

2. Generate rsa_private key (with password WTP_PASSWORD, 'client' for example) for client.
```
openssl genrsa -aes256 -out client_rsa_private.key 2048
```

3. Generate rsa_private key (with password AC_PASSWORD, 'server' for example) for server.
```
openssl genrsa -aes256 -out server_rsa_private.key 2048
```

## Step 2 Generate self-signed root certificate

Self-signed root certificate is used to issue pem for client & server later.

1. Generate 10 years valid self-signed root certificate with ca_rsa_private.key, we will be asked to input CA_PASSWORD('prova' for example), and answer a list of question like this.
```
openssl req -new -x509 -days 3650 -key ca_rsa_private.key -out ca.pem
```
Enter pass phrase for ca_rsa_private.key: *Enter ca_rsa_private.key's password(CA_PASSWORD)*
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
Country Name (2 letter code) [AU]:**CN**
State or Province Name (full name) [Some-State]:**China**
Locality Name (eg, city) []:**beijing**
Organization Name (eg, company) [Internet Widgits Pty Ltd]:**Liteon**
Organizational Unit Name (eg, section) []:**SLA**
Common Name (e.g. server FQDN or YOUR name) []:**CAPWAP_CA**
Email Address []:**ca@localhost**

## Step 3 Generate client/server side csr with correct EKU(Extended Key Usage)

During generating csr with correct EKU, we need additional individual config files for both client(WTP) & server(AC)

Config file for WTP
```
frankzhou@fx:~/capwap/src$ cat wtp_config
[ req ]
distinguished_name	= req_distinguished_name
req_extensions = v3_req # The extensions to add to a certificate request

[ req_distinguished_name ]
countryName						= Country Name (2 letter code)
countryName_default				= CN
countryName_min					= 2
countryName_max					= 2

stateOrProvinceName				= State or Province Name (full name)
stateOrProvinceName_default		= China

localityName					= Locality Name (eg, city)
localityName_default			= beijing

0.organizationName				= Organization Name (eg, company)
0.organizationName_default		= Liteon


organizationalUnitName			= Organizational Unit Name (eg, section)
organizationalUnitName_default	= SLA

commonName				= Common Name (e.g. server FQDN or YOUR name)
commonName_max			= 64

emailAddress			= Email Address
emailAddress_max		= 64

[ v3_req ]
#basicConstraints=CA:FALSE
extendedKeyUsage = critical,1.3.6.1.5.5.7.3.19
```

Config file for AC
```
frankzhou@fx:~/capwap/src$ cat ac_config
[ req ]
distinguished_name	= req_distinguished_name
req_extensions = v3_req # The extensions to add to a certificate request

[ req_distinguished_name ]
countryName						= Country Name (2 letter code)
countryName_default				= CN
countryName_min					= 2
countryName_max					= 2

stateOrProvinceName				= State or Province Name (full name)
stateOrProvinceName_default		= China

localityName					= Locality Name (eg, city)
localityName_default			= beijing

0.organizationName				= Organization Name (eg, company)
0.organizationName_default		= Liteon


organizationalUnitName			= Organizational Unit Name (eg, section)
organizationalUnitName_default	= SLA

commonName				= Common Name (e.g. server FQDN or YOUR name)
commonName_max			= 64

emailAddress			= Email Address
emailAddress_max		= 64

[ v3_req ]
#basicConstraints=CA:FALSE
extendedKeyUsage = critical,1.3.6.1.5.5.7.3.18
```

**Notice**
The **extendedKeyUsage** of two file are different at the bottom of the file.

1. Command to generate client csr, you need to input WTP_PASSWORD('client' for example) and answer a series of question.
```
openssl req -new -key client_rsa_private.key -out client.csr -config wtp_config
```
Enter pass phrase for client_rsa_private.key:*Enter client_rsa_private.key's password(WTP_PASSWORD)*
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
Country Name (2 letter code) [CN]:**Enter**
State or Province Name (full name) [China]:**Enter**
Locality Name (eg, city) [beijing]:**Enter**
Organization Name (eg, company) [Liteon]:**Enter**
Organizational Unit Name (eg, section) [SLA]:**Enter**
Common Name (e.g. server FQDN or YOUR name) []:**CAPWAP_WTP**
Email Address []:**wtp@localhost**

2. Command to generate server csr, you need to input AC_PASSWORD('server' for example) and answer a series of question.
```
openssl req -new -key server_rsa_private.key -out server.csr -config ac_config
```
Enter pass phrase for server_rsa_private.key:*Enter server_rsa_private.key's password(AC_PASSWORD)*
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
Country Name (2 letter code) [CN]:**Enter**
State or Province Name (full name) [China]:**Enter**
Locality Name (eg, city) [beijing]:**Enter**
Organization Name (eg, company) [Liteon]:**Enter**
Organizational Unit Name (eg, section) [SLA]:**Enter**
Common Name (e.g. server FQDN or YOUR name) []:**CAPWTP_AC**
Email Address []:**ac@localhost**

## Step 4 Sign client/server request(*.csr) with correct EKU using CA's self-signed pem & ca_rsa_private.key

Here we need additional extension file during assigning request:
```
frankzhou@fx:~/capwap/src$ cat wtp_extfile
#x509_extensions	= usr_cert

#[ usr_cert ]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer

extendedKeyUsage = critical,clientAuth,1.3.6.1.5.5.7.3.19
frankzhou@fx:~/capwap/src$ cat ac_extfile
#x509_extensions	= usr_cert

#[ usr_cert ]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer

extendedKeyUsage = critical,serverAuth,1.3.6.1.5.5.7.3.18

```

1. Sign 10 years life long server request with correct EKU by CA's self-signed pem & ca_rsa_private.key
```
openssl x509 -req -days 3650 -in server.csr -CA ca.pem -CAkey ca_rsa_private.key -CAcreateserial -extfile ac_extfile -out ac.pem
```
Signature ok
subject=/C=CN/ST=China/L=beijing/O=Liteon/OU=SLA/CN=CAPWTP_AC/emailAddress=ac@localhost
Getting CA Private Key
Enter pass phrase for ca_rsa_private.key:*Enter ca_rsa_private.key's password(CA_PASSWORD)*

2. Sign 10 years life long client request with correct EKU by CA's self-signed pem & ca_rsa_private.key
```
openssl x509 -req -days 3650 -in client.csr -CA ca.pem -CAkey ca_rsa_private.key -CAcreateserial -extfile wtp_extfile -out wtp.pem
```
Signature ok
subject=/C=CN/ST=China/L=beijing/O=Liteon/OU=SLA/CN=CAPWAP_WTP/emailAddress=wtp@localhost
Getting CA Private Key
Enter pass phrase for ca_rsa_private.key:*Enter ca_rsa_private.key's password(CA_PASSWORD)*

## Step 5 Concatenate pem & private key into a single file for client/server

1. Integrate client side private key & pem into a single file:
```
cat wtp.pem client_rsa_private.key > client.pem
```

2. Integrate server side private key & pem into a single file:
```
cat ac.pem server_rsa_private.key > server.pem
```

3. Rename ca.pem into 'root.pem' which we are using.

