# Certificate Generator

Leverages OpenSSL to generate self-signed certificates for device certificate testing purposes.  This repo is meant to be run from a bash terminal.  If you're using Windows, you can use the Windows Subsystem for Linux (WSL) to run the scripts.

This sample is based heavily on [OpenSSL CA](https://openssl-ca.readthedocs.io/) docs and the blog [post](https://kevinsaye.wordpress.com/2022/04/12/using-openssl-to-create-edge-device-ca-certificates-for-azure-iot-edge/) by Kevin Saye.

## Usage

1. Create the `certs` directory and initialize the files:
```bash
mkdir -p certs/newcerts
touch certs/index.txt
echo 1000 > certs/serial
```

1. Generate the Root CA:
```bash
# Generate the Root CA Private Key
openssl genrsa -out certs/rootCAPrivateKey.pem 4096
 
# Generate the Root CA Public Key
openssl req -new -x509 -config openssl-root.cnf -nodes -days 7300 -key certs/rootCAPrivateKey.pem -out certs/rootCAPublicKey.pem -subj "/CN=Demo Root CA" -extensions "v3_ca"
```

1. Create an Intermediate CA:
```bash
# Generate the Issuing CA Private Key and CSR
openssl req -newkey rsa:4096 -nodes -keyout certs/intermediateCAPrivateKey.pem -out certs/issuingCertificateRequest.csr -subj "/CN=Demo Intermediate CA"

# Generate the Issuing CA Public Key by signing the CSR from the Root CA
openssl ca -batch -config openssl-intermediate.cnf -in certs/issuingCertificateRequest.csr -days 3650 -cert certs/rootCAPublicKey.pem -keyfile certs/rootCAPrivateKey.pem -keyform PEM -out certs/intermediateCAPublicKey.pem -extensions v3_intermediate_ca
```

1. Create an Intermediate CA Chain:
```bash
# Create the Intermediate CA Chain
cat certs/intermediateCAPublicKey.pem certs/rootCAPublicKey.pem > certs/intermediateCAPublicKeyChain.pem
```

1. Create a Device Certificate:
```bash
# Set the device name
DEVICENAME=Device1

# Generate the Device Private Key and CSR
openssl req -newkey rsa:4096 -nodes -keyout certs/${DEVICENAME}PrivateKey.pem -out certs/${DEVICENAME}CertificateRequest.csr -subj "/CN=${DEVICENAME}"

# Generate the Device Public Key by signing the CSR from the Intermediate CA
openssl ca -batch -config openssl-intermediate.cnf -in certs/${DEVICENAME}CertificateRequest.csr -days 365 -cert certs/intermediateCAPublicKey.pem -keyfile certs/intermediateCAPrivateKey.pem -keyform PEM -out certs/${DEVICENAME}PublicKey.pem -extensions usr_cert
```

1. Create a Device Certificate Chain:
```bash
# Create the Device Certificate Chain
cat certs/${DEVICENAME}PublicKey.pem certs/intermediateCAPublicKey.pem certs/rootCAPublicKey.pem > certs/${DEVICENAME}PublicKeyChain.pem
```

1. Get the fingerprint of the cert.
```bash
# Get the raw fingerprint of the cert.
openssl x509 -in certs/${DEVICENAME}PublicKey.pem -outform DER | openssl dgst -sha1
```