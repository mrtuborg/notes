---
title: "Pki Openssl"
date: 2023-07-04T10:39:20+02:00
draft: true
---

# This document describes how to install and configure PKI infrastructure based on openssl.

Prerequisites:
    - openssl
    - WEB server for CRL publishing

## Contents
* Create generic OpenSSL configuration
* Create Root CA
* Create Server CA (Intermediate CA)
* Create Client CA (Intermediate CA)
* Create p12 for client
* Generate CRL

## Create generic OpenSSL configuration

### Step 1. Prepare environment

```bash
mkdir .ca
cd .ca
mkdir certs crl reqs private
touch index.txt
echo 'unique_subject = yes' > index.txt.attr
echo 01 > serial
echo 01 > crlnumber
```

### Step 2. Create openssl.cnf

```bash
cat << EOF > openssl.cnf
HOME                   = .
RANDFILE               = \$ENV::HOME/.rnd

[ ca ]
default_ca = CA_default       # The default ca section

[ CA_default ]
dir = .                       # top dir, where everything is kept
certs = \$dir/certs           # where the issued certs are kept
crl_dir = \$dir/crl           # where the issued crl are kept
database = \$dir/index.txt    # database index file.
#unique_subject = no          # Set to 'no' to allow creation of
                              # several ctificates with same subject.
new_certs_dir = \$dir/certs   # default place for new certs.

certificate = \$dir/certs/root.cer   # The CA certificate
serial = \$dir/serial         # The current serial number
crlnumber = \$dir/crlnumber   # the current crl number
                              # must be commented out to leave a V1 CRL
crl = \$dir/crl/root.crl      # The current CRL
private_key = \$dir/private/root.key # The private key
RANDFILE = \$dir/private/.rand       # private random number file

x509_extensions = client_cert    # The extensions to add to the cert
                                # basicConstraints = CA:FALSE
                                # nsCertType = client, server, email, objsign
                                # keyUsage = digitalSignature, keyEncipherment
                                # extendedKeyUsage = clientAuth, serverAuth
                                # nsComment = "OpenSSL Generated Certificate"

# Comment out the following two lines for the "traditional"
# (and highly broken) format.
name_opt = ca_default           # Subject Name options
cert_opt = ca_default           # Certificate field options

# Extension copying option: use with caution.
# copy_extensions = copy

# Extensions to add to a CRL. Note: Netscape communicator chokes on V2 CRLs
# so this is commented out by default to leave a V1 CRL.
# crlnumber must also be commented out to leave a V1 CRL.
crl_extensions = crl_ext

default_days = 365              # how long to certify for
default_crl_days = 30           # how long before next CRL
default_md = sha256             # use public key default MD
preserve = no                   # keep passed DN ordering

# A few difference way of specifying how similar the request should look
# For type CA, the listed attributes must be the same, and the optional
# and supplied fields are just that :-)
policy = policy_match

# For the CA policy
[ policy_match ]
countryName = match
organizationName = supplied
organizationalUnitName = optional
commonName = supplied
emailAddress = optional

# For the 'anything' policy
# At this point in time, you must list all acceptable 'object'
# types.
[ policy_anything ]
countryName = optional
stateOrProvinceName = optional
localityName = optional
organizationName = optional
organizationalUnitName = optional
commonName = supplied
emailAddress = optional

[ req ]
default_bits = 2048
default_keyfile = privkey.pem
distinguished_name = req_distinguished_name
attributes = req_attributes
x509_extensions = v3_ca # The extensions to add to the self signed cert

# This sets a mask for permitted string types. There are several options.
# default: PrintableString, T61String, BMPString.
# pkix   : PrintableString, BMPString.
# utf8only: only UTF8Strings.
# nombstr : PrintableString, T61String (no BMPStrings or UTF8Strings).
# MASK:XXXX a literal mask value.
# WARNING: current versions of Netscape crash on BMPStrings or UTF8Strings
# so use this option with caution!
string_mask = utf8only

[ req_distinguished_name ]
countryName = Country Name (2 letter code)
countryName_default = NO
countryName_min = 2
countryName_max = 2

0.organizationName = Organization Name (eg, company)
0.organizationName_default = My Company

commonName = Common Name (e.g. server FQDN or YOUR name)
commonName_max = 64

[ req_attributes ]
challengePassword = A challenge password
challengePassword_min = 4
challengePassword_max = 20

unstructuredName = An optional company name

[ server_cert ]
basicConstraints = CA:FALSE

# This is typical in keyUsage for a server certificate.
keyUsage = digitalSignature, keyEncipherment

# Set EKU
extendedKeyUsage = serverAuth

# PKIX is typical in keyUsage for a server certificate.
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer

# This stuff is for subjectAltName and issuerAltname.
subjectAltName = @subjectAltName_server

# Copy subject details
# issuerAltName=issuer:copy

# authorityInfoAccess = caIssuers;URI:http://www.example.com/root.cer
# we don't need AIA for second level certificates
crlDistributionPoints = URI:http://www.example.com/root.crl

[ subjectAltName_server ]
DNS = $ENV::DNSNAME

[ client_cert ]

# These extensions are added when 'ca' signs a request.

# This goes against PKIX guidelines but some CAs do it and some software
# requires this to avoid interpreting an end user certificate as a CA.

basicConstraints = CA:FALSE

# This is typical in keyUsage for a client certificate.
keyUsage = digitalSignature, keyEncipherment

# Set EKU
extendedKeyUsage = clientAuth

# PKIX is typical in keyUsage for a server certificate.
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer

# This stuff is for subjectAltName and issuerAltname.
subjectAltName = @subjectAltName_client

# Copy subject details
# issuerAltName=issuer:copy

# authorityInfoAccess = caIssuers;URI:http://www.example.com/root.cer
# we don't need AIA for second level certificates
crlDistributionPoints = URI:http://www.example.com/root.crl

[ subjectAltName_client ]
DNS = $ENV::EMAIL

[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer:always

basicConstraints = critical, CA:true, pathlen:0
keyUsage = cRLSign, keyCertSign

basicConstraints = CA:true
keyUsage = digitalSignature, cRLSign, keyCertSign

[ crl_ext ]
# CRL extensions.
# Only issuerAltName and authorityKeyIdentifier make any sense in a CRL.
# issuerAltName=issuer:copy
authorityKeyIdentifier=keyid:always,issuer:always
EOF
```

## 2. Create Root CA

### Step 1. Create .req file (omit password)

```bash
$ openssl req -config mycompany.cnf -new -sha256 -newkey rsa:2048 -nodes -keyout private/MyCompany-Root.key -out reqs/MyCompany-Root.req
...
Common Name (eg, YOUR name) []:MyCompany Root CA
...

```

### Step 2. Create self-signed root certificate

```bash
$ openssl req -config mycompany.cnf -in reqs/MyCompany-Root.req -days 7300 -x509 -key private/MyCompany-Root.key -out/MyCompany-Root.cer
```

## 3. Create server certificate request

### Step 1. Create .req file (omit password, use ipsec.mycompany.no as CN)

```bash
$ openssl req -config mycompany.cnf -new -newkey rsa:2048 -nodes -keyout private/server.key -out reqs/server.req
...
Common Name (eg, YOUR name) []:ipsec.mycompany.no
...
```

### Step 2. Define Subject Alternative Name in variable

```bash
$ export DNSNAME=ipsec.mycompany.no
```

### Step 3. Confirm issuing of certificate two times

```bash
Sign the certificate? [y/n]: y

1 out of 1 certificate requests certified, commit? [y/n] y
Write out database with 1 new entries
Data Base Updated
```

## 4. Create client certificate request

### Step 1. Create .req file (omit password, use static CN for clients)

```bash
$ openssl req -config clustertech.cnf -new -newkey rsa:2048 -nodes -keyout private/client.key -out reqs/client.req
...
Common Name (eg, company) []:IPSec Client Auth
...
```

### Step 2. Define auth e-mail in variable

```bash
$ export EMAIL=test@mycompany.no
```

### Step 3. Sign client certificate
SAN e-mail will be used as user ID during Radius authorization

```bash
$ openssl ca -config mycompany.cnf -days 365 -extensions client_cert -in reqs/client.req -out certs/client.cer
```

### Step 4. Confirm issuing of certificate two times

```bash
Sign the certificate? [y/n]: y

1 out of 1 certificate requests certified, commit? [y/n] y
Write out database with 1 new entries
Data Base Updated
```

## 5. Create p12 for client

Create pkcs#12 file for client
Use strong password for p12 file, do not send password together with p12 file!
```bash
$ openssl pkcs12 -export -in certs/client.cer -inkey private/client.key -out certs/client.p12
```

## 6. Generate CRL

### Step 1. Create CRL

```bash
$ openssl ca -config mycompany.cnf -gencrl -out crl/root.crl
```

### Step 2. Place CRL

Copy crl/root.crl to WEB server directory, where it can be accessed by clients
```bash
$ cp crl/root.crl /var/www/html/
```
