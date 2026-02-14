# Certificate Generator

Leverages OpenSSL to generate self-signed P-384 ECDSA certificates for device certificate testing purposes. This repo is meant to be run from a bash terminal. If you're using Windows, you can use the Windows Subsystem for Linux (WSL) to run the scripts.

## Usage

1. Create the `certs` directory:
```bash
mkdir -p certs
```

2. Generate the Root CA:
```bash
# Generate the Root CA Private Key
openssl ecparam -genkey -name secp384r1 -noout -out certs/rootCAPrivateKey.pem

# Generate the Root CA Public Key
openssl req -new -x509 -sha384 -days 7300 -key certs/rootCAPrivateKey.pem \
  -out certs/rootCAPublicKey.pem -subj "/CN=Demo Root CA" \
  -addext "basicConstraints=critical,CA:true" \
  -addext "keyUsage=critical,digitalSignature,cRLSign,keyCertSign"
```

3. Create an Intermediate CA:
```bash
# Generate the Issuing CA Private Key
openssl ecparam -genkey -name secp384r1 -noout -out certs/intermediateCAPrivateKey.pem

# Generate the Issuing CA CSR
openssl req -new -sha384 -key certs/intermediateCAPrivateKey.pem \
  -out certs/issuingCertificateRequest.csr -subj "/CN=Demo Intermediate CA"

# Generate the Issuing CA Public Key by signing the CSR from the Root CA
openssl x509 -req -sha384 -days 3650 -in certs/issuingCertificateRequest.csr \
  -CA certs/rootCAPublicKey.pem -CAkey certs/rootCAPrivateKey.pem -CAcreateserial \
  -out certs/intermediateCAPublicKey.pem \
  -extfile <(printf "basicConstraints=critical,CA:true,pathlen:0\nkeyUsage=critical,digitalSignature,cRLSign,keyCertSign")
```

4. Create an Intermediate CA Chain:
```bash
# Create the Intermediate CA Chain
cat certs/intermediateCAPublicKey.pem certs/rootCAPublicKey.pem > certs/intermediateCAPublicKeyChain.pem
```

5. Create a Device Certificate:
```bash
# Set the device name
DEVICENAME=Device1

# Generate the Device Private Key
openssl ecparam -genkey -name secp384r1 -noout -out certs/${DEVICENAME}PrivateKey.pem

# Generate the Device CSR
openssl req -new -sha384 -key certs/${DEVICENAME}PrivateKey.pem \
  -out certs/${DEVICENAME}CertificateRequest.csr -subj "/CN=${DEVICENAME}"

# Generate the Device Public Key by signing the CSR from the Intermediate CA
openssl x509 -req -sha384 -days 365 -in certs/${DEVICENAME}CertificateRequest.csr \
  -CA certs/intermediateCAPublicKey.pem -CAkey certs/intermediateCAPrivateKey.pem -CAcreateserial \
  -out certs/${DEVICENAME}PublicKey.pem \
  -extfile <(printf "basicConstraints=CA:FALSE\nkeyUsage=critical,nonRepudiation,digitalSignature,keyAgreement\nextendedKeyUsage=clientAuth")
```

6. Create a Device Certificate Chain:
```bash
# Create the Device Certificate Chain
cat certs/${DEVICENAME}PublicKey.pem certs/intermediateCAPublicKey.pem certs/rootCAPublicKey.pem > certs/${DEVICENAME}PublicKeyChain.pem
```

7. Get the fingerprint of the cert:
```bash
# Get the raw fingerprint of the cert.
openssl x509 -in certs/${DEVICENAME}PublicKey.pem -outform DER | openssl dgst -sha1
```

8. Print a PEM file as a double quoted line with `\n` (useful for JSON, environment variables, etc.):
```bash
# Print a flattened certificate
awk 'BEGIN{printf "\""} NF {sub(/\r/, ""); printf "%s\\n",$0;} END{print "\""}' certs/${DEVICENAME}PublicKey.pem

# Print a flattened private key
awk 'BEGIN{printf "\""} NF {sub(/\r/, ""); printf "%s\\n",$0;} END{print "\""}' certs/${DEVICENAME}PrivateKey.pem
```