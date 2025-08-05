# mTLS Certificate Generation

## Overview

This document describes how to generate mTLS certificates for the job worker service. Certificates expire after 15 days to demonstrate proper certificate lifecycle management in a demo environment.

## Manual Generation

### Prerequisites

- OpenSSL installed
- `certs/` directory in project root

### Step-by-Step Commands

```bash
# Create certs directory
mkdir -p certs
cd certs

# 1. Create Certificate Authority (CA)
openssl genrsa -out ca.key 4096
openssl req -new -x509 -key ca.key -sha256 -subj "/CN=JobWorker-CA" -days 15 -out ca.crt

# 2. Generate Server Certificate
openssl genrsa -out server.key 4096
openssl req -new -key server.key -out server.csr -subj "/CN=localhost"
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -out server.crt -days 15 -sha256 -CAcreateserial

# 3. Generate Client Certificates
# Client 1 (user1)
openssl genrsa -out client1.key 4096
openssl req -new -key client1.key -out client1.csr -subj "/CN=user1"
openssl x509 -req -in client1.csr -CA ca.crt -CAkey ca.key -out client1.crt -days 15 -sha256

# Client 2 (user2)
openssl genrsa -out client2.key 4096
openssl req -new -key client2.key -out client2.csr -subj "/CN=user2"
openssl x509 -req -in client2.csr -CA ca.crt -CAkey ca.key -out client2.crt -days 15 -sha256

# Clean up CSR files
rm *.csr
```

## Generated Files

After generation, the `certs/` directory will contain:

```
certs/
├── ca.crt          # Certificate Authority certificate
├── ca.key          # CA private key
├── ca.srl          # CA serial number file
├── server.crt      # Server certificate
├── server.key      # Server private key
├── client1.crt     # Client certificate for user1
├── client1.key     # Client private key for user1
├── client2.crt     # Client certificate for user2
└── client2.key     # Client private key for user2
```

## Certificate Details

- **Validity**: 15 days from generation date
- **Key Size**: 4096-bit RSA
- **Hash Algorithm**: SHA-256
- **Server CN**: localhost (for local testing)
- **Client CNs**: user1, user2 (for user identification)

## Script Implementation

For automated generation, create a script with the following characteristics:

### Script Requirements

- **Location**: `scripts/generate-certs.sh`
- **Permissions**: Executable (`chmod +x`)
- **Hardcoded paths**: All paths relative to project root
- **Error handling**: Exit on any command failure
- **Directory creation**: Automatically create `certs/` directory
- **Cleanup**: Remove intermediate CSR files

### Script Structure

```bash
#!/bin/bash
set -e  # Exit on any error

# Hardcoded parameters
CERT_DIR="certs"
CA_DAYS=15
CERT_DAYS=15
KEY_SIZE=4096

# Create directory and navigate
mkdir -p "$CERT_DIR"
cd "$CERT_DIR"

# Generate certificates (commands from manual section)
# ... certificate generation commands ...

# Cleanup
rm -f *.csr

echo "Certificates generated successfully"
echo "Valid for $CERT_DAYS days from $(date)"
```

## Usage Notes

- **Demo Purpose**: 15-day expiration demonstrates certificate lifecycle
- **Local Testing**: Server certificate uses CN=localhost
- **User Identity**: Client certificate CN field used for job ownership
- **Security**: Private keys should not be committed to version control
- **Regeneration**: Run script again when certificates expire

## Integration with Application

The application expects certificates in the following locations:

```go
// Hardcoded paths in application
serverCert := "certs/server.crt"
serverKey := "certs/server.key"
caCert := "certs/ca.crt"
clientCert := "certs/client1.crt"
clientKey := "certs/client1.key"
```

## Verification

To verify generated certificates:

```bash
# Check certificate details
openssl x509 -in certs/server.crt -text -noout
openssl x509 -in certs/client1.crt -text -noout

# Verify certificate chain
openssl verify -CAfile certs/ca.crt certs/server.crt
openssl verify -CAfile certs/ca.crt certs/client1.crt
```