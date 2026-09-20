# OpenSSL-PKI-Workflow
 OpenSSL Public Key Infrastructure (PKI) Setup &amp; Operations  This guide documents the end-to-end lifecycle of a local Certificate Authority (CA), including Root CA initialization, server CSR generation, certificate signing, verification, and key escrow.

# OpenSSL PKI Operations Guide

This repository contains operational documentation and configuration files for running a single-tier Public Key Infrastructure (PKI) using OpenSSL. It enforces a strict operational boundary between the **CA System** (Offline / Secure Node) and the **Server System** (Web / Host Node).

---

## 🏗 Architecture & File Layout

```
.
├── config/
│   └── openssl.cnf          # CA configuration file (database paths, policy, extensions)
├── ca/
│   ├── cakey.pem            # Root CA Private Key (AES-256 encrypted)
│   ├── cacert.pem           # Root CA Certificate (Trust Anchor)
│   ├── index.txt            # OpenSSL CA database tracking file
│   ├── serial               # Certificate serial counter file
│   └── newcerts/            # Signed certificate history archive
├── server/
│   ├── www.key              # Web Server Private Key (unencrypted)
│   ├── www.csr              # Certificate Signing Request
│   ├── www.pem              # Issued X.509 Certificate
│   ├── www.key.bak          # Encrypted Private Key Backup
│   └── www.der              # Binary DER Certificate
└── screenshots/             # Workflow execution screenshots

```

---

## 🚀 Step-by-Step PKI Workflow

---

### Phase 1: Root CA Configuration & Initialization

#### Step 1.1: Generate the Root CA RSA Key Pair

Generates a secure RSA private key encrypted on disk for the Root CA anchor.

* **Execution Side:** `CA SIDE`
* **Command Switches:**
* `genrsa` — Selects the RSA key generation engine.
* `-aes256` — Applies AES-256 symmetric cipher encryption.
* `-out` — Specifies the private key output file.
* `4096` — Sets key size to 4096 bits.



```bash
openssl genrsa -aes256 -out ca/cakey.pem 4096

```

---

#### Step 1.2: Generate the Self-Signed Root X.509 Certificate

Creates the public self-signed Root CA certificate that acts as the organization's Trust Anchor.

* **Execution Side:** `CA SIDE`
* **Command Switches:**
* `req` — Calls the certificate request and generation tool.
* `-config` — Specifies the OpenSSL configuration file path.
* `-key` — Identifies the signing private key.
* `-new -x509` — Generates a self-signed X.509 certificate instead of a CSR.
* `-days` — Defines the validity period in days (`7300` = 20 years).
* `-sha256` — Enforces the SHA-256 signature hash algorithm.
* `-out` — Specifies the public certificate output file.



```bash
openssl req -config config/openssl.cnf -key ca/cakey.pem -new -x509 -days 7300 -sha256 -out ca/cacert.pem

```

---

### Phase 2: Server Key Pair & CSR Generation

#### Step 2.1: Generate Web Server Key Pair and CSR

Generates a web server keypair alongside a Certificate Signing Request (CSR).

* **Execution Side:** `SERVER SIDE`
* **Command Switches:**
* `req` — Calls the certificate request utility.
* `-nodes` — Skips passphrase encryption ("No DES") to allow automated service restarts.
* `-new` — Indicates a new certificate signing request creation.
* `-newkey` — Generates a new RSA private key of the specified length (`rsa:2048`).
* `-out` — Specifies the output path for the CSR file.
* `-keyout` — Specifies the output path for the generated private key file.



```bash
openssl req -nodes -new -newkey rsa:2048 -out server/www.csr -keyout server/www.key

```

---

### Phase 3: Certificate Issuance & Transmission

#### Step 3.1: Transmit CSR to CA Server

Securely transfers the CSR file to the CA node. The private key (`www.key`) stays on the server.

* **Execution Side:** `SERVER SIDE` $\rightarrow$ `CA SIDE`

```bash
scp server/www.csr user@ca-server:/path/to/pki/server/www.csr

```

---

#### Step 3.2: Sign the CSR using the CA Server

Evaluates the CSR, applies server extensions, signs the certificate, and logs it to the CA database.

* **Execution Side:** `CA SIDE`
* **Command Switches:**
* `ca` — Launches the OpenSSL CA database and signing engine.
* `-config` — Points to the CA configuration file.
* `-extensions` — Selects the X.509 v3 extension section (`webserver`).
* `-infiles` — Identifies the incoming CSR file to sign.
* `-out` — Defines the path for the issued X.509 certificate.



```bash
openssl ca -config config/openssl.cnf -extensions webserver -infiles server/www.csr -out server/www.pem

```

---

#### Step 3.3: Return Signed Certificate to Server

Transfers the issued public certificate back to the web host.

* **Execution Side:** `CA SIDE` $\rightarrow$ `SERVER SIDE`

```bash
scp server/www.pem user@web-server:/path/to/pki/server/www.pem

```

---

### Phase 4: Inspection & Chain Verification

#### Step 4.1: View Certificate Text Structure

Decodes and inspects the human-readable text contents of the X.509 certificate.

* **Execution Side:** `SERVER SIDE`
* **Command Switches:**
* `x509` — Calls the X.509 display and management utility.
* `-noout` — Suppresses the Base64 PEM encoded output block.
* `-text` — Prints the decoded certificate fields in plain text.
* `-in` — Specifies the target certificate file to parse.



```bash
openssl x509 -noout -text -in server/www.pem

```

---

#### Step 4.2: Verify Certificate Trust Chain

Validates the cryptographic trust chain of the issued certificate against the Root CA anchor.

* **Execution Side:** `SERVER SIDE`
* **Command Switches:**
* `verify` — Launches the certificate chain validation tool.
* `-verbose` — Prints detailed verification processing messages.
* `-cafile` — Points to the trusted Root CA file to validate against.



```bash
openssl verify -verbose -cafile ca/cacert.pem server/www.pem

```

---

### Phase 5: Key Escrow, Backup & Format Conversions

#### Step 5.1: Encrypt Private Key for Secure Escrow

Encrypts an unencrypted server private key prior to off-site backup or vault escrow.

* **Execution Side:** `SERVER SIDE`
* **Command Switches:**
* `rsa` — Invokes the RSA key processing utility.
* `-aes256` — Applies AES-256 cipher encryption with a passphrase.
* `-in` — Specifies the source unencrypted private key file.
* `-out` — Specifies the destination encrypted backup file.



```bash
openssl rsa -aes256 -in server/www.key -out server/www.key.bak

```

---

#### Step 5.2: Convert PEM Certificate to Binary DER Format

Converts an ASCII Base64 `.pem` certificate into binary `.der` format for legacy application environments.

* **Execution Side:** `SERVER SIDE`
* **Command Switches:**
* `x509` — Calls the X.509 structure converter.
* `-outform` — Specifies output encoding format (`der` for binary ASN.1).
* `-in` — Specifies the input ASCII PEM file.
* `-out` — Specifies the output binary DER file.



```bash
openssl x509 -outform der -in server/www.pem -out server/www.der

```

---
