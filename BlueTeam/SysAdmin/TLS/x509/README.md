# X.509 Certificates and Keys

## The Overarching SSL Process

Before diving into certificates, understand the full picture of how SSL works end to end.

### The 10 Steps of SSL

1. The **Certificate Authority (CA)** generates its own public/private key pair
2. The CA creates a **self-signed certificate** — its own identity document, signed by its own private key
3. A **server** (e.g. blue.com) generates its own public/private key pair
4. The server generates a **CSR (Certificate Signing Request)** containing its public key, signed by its own private key
5. The server sends the CSR to the CA
6. The CA **validates** that the server is who it claims to be (e.g. actually owns blue.com)
7. The CA generates a **certificate** using info from the CSR, including the server's public key
8. The CA **signs the certificate** with its own private key
9. The CA sends the signed certificate back to the server
10. The server presents this certificate to clients so they can connect securely

> Key insight: The server's public key travels from the server → CSR → certificate. The CA never sees or touches the server's private key.

---

## What Is Inside a Certificate

Certificates follow the **X.509 standard**, which defines three sections:

```
[ Certificate Data       ]  ← all the important fields
[ Signature Algorithm    ]  ← hashing algo + key type used to sign
[ Signature              ]  ← hash of Certificate Data, encrypted with CA's private key
```

The signature algorithm field is **duplicated inside Certificate Data** intentionally — because the hash only covers Certificate Data, so if it wasn't duplicated there, the signature algorithm field outside could be tampered with undetected.

---

### Fields Inside Certificate Data

#### Version Number
- Stored as a hex value
- `0x00` = X.509 v1 (obsolete)
- `0x01` = X.509 v2 (obsolete)
- `0x02` = X.509 v3 (current standard) ✅
- **Not** the same as the TLS/SSL version — these are completely separate things

#### Serial Number
- A 20-byte number that uniquely identifies a certificate **within a specific CA**
- Two different CAs can issue certificates with the same serial number — that's fine
- Used to check revocation status by querying the issuing CA (via CRL or OCSP)

#### Signature Algorithm
- Specifies the hashing algorithm used on Certificate Data
- Specifies the asymmetric key type used to sign the digest
- **Avoid** (insecure): `MD5withRSA`, `SHA1withRSA`
- **Use** (secure): `SHA256withRSA`, `SHA256withECDSA`

#### Validity
- Two values: **Not Before** and **Not After**
- Certificates can be invalid in both directions — too old AND too new
- Example: a cert valid March 2013 – March 2014 would be rejected in both 2012 and 2015

#### Subject and Issuer
Both fields use **Distinguished Name (DN) format** — a hierarchy of attribute-value pairs:

| Attribute | Abbreviation | Example |
|---|---|---|
| Country | C | US, GB, RU |
| State | ST | California |
| Locality (City) | L | San Francisco |
| Organization | O | Reddit Inc |
| Common Name | CN | *.reddit.com |

- **Subject DN** = identity of the server the cert protects
- **Issuer DN** = identity of the CA that signed the cert
- If Subject DN == Issuer DN → **self-signed certificate** (typical for CA root certs)
- **Common Name** is the only mandatory attribute — the browser compares this to the URL bar

##### Wildcard Certificates
- A `*` in the CN matches **exactly one subdomain level**
- `*.google.com` matches `mail.google.com` ✅
- `*.google.com` does NOT match `us.mail.google.com` ❌ (two levels deep)
- `*.google.com` does NOT match `google.com` ❌ (wildcard requires something to be there)

#### Public Key
Presented as two values:

**RSA key:**
- `Modulus` = the n value (product of two large primes p × q)
- `Exponent` = the e value (public key exponent)

**ECDSA key:**
- `Public Key Value`
- `Curve Name` (e.g. prime256v1)

#### Extensions
- Only exist in X.509 v3 certificates
- Optional fields that add features or restrictions
- The reason there is no X.509 v4 — extensions allow the standard to grow without versioning

---

## Certificate Extensions

### Key Usage
Sets limits on what the certificate's keys can be used for by **purpose**:
- Encrypt data
- Establish symmetric keys (key exchange)
- Sign certificates
- Verify signatures
- Verify CRL signatures

### Extended Key Usage (EKU)
Sets limits by **protocol or role**:
- SSL Server
- SSL Client
- Email
- IPsec

> Why limit keys? Reduces failure domain — if a key is compromised, it can only be misused for its stated purpose.

### Basic Constraints
- Determines whether the subject is a **CA** or an **End Entity**
- CA certificates can issue other certificates; end entity certificates cannot
- Prevents a regular website cert from being used to sign other certs

### Name Constraints
- Limits a CA to only signing certificates for a specific domain
- Common for internal corporate CAs — e.g. Acme Corp's CA can only sign `*.acme.com`

### Subject Key Identifier / Authority Key Identifier
- Unique identifiers for specific public keys
- **Subject Key Identifier** = identifies the subject's public key
- **Authority Key Identifier** = identifies which CA key signed this cert
- Used to track keys across certificate renewals (you can reuse the same key pair with a new cert)
- If both identifiers are **identical** → self-signed certificate

### Subject Alternative Name (SAN)
- Allows one certificate to protect **multiple different domains**
- Unlike wildcards, SANs don't need to share the same root domain
- Example: one cert covering `*.live.com`, `hotmail.com`, and `*.hotmail.com`
- Most modern certs use this extension

### CRL Distribution Point
- Provides the URL to download the Certificate Revocation List for this cert's CA

### Authority Information Access
Two things:
1. URL to download the CA's own certificate (so clients can verify the chain)
2. OCSP responder location (for real-time revocation checking)

---

## What Is Inside a Private Key File

A private key file (PEM format) contains a set of values used in asymmetric math:

| Field Name | What It Maps To |
|---|---|
| Modulus | n = p × q |
| Public Exponent | e (public key) |
| Private Exponent | d (private key) |
| Prime 1 | p |
| Prime 2 | q |
| Exponent 1, 2 | Used for efficient decryption (Chinese Remainder Theorem) |
| Coefficient | Also used for efficient decryption |

### Matching a Certificate to Its Private Key
The **Modulus** value is present in both the certificate (as part of the public key) and the private key file. Matching certificates and keys have **identical modulus values**.

OpenSSL commands to check:
```bash
# Get modulus from certificate
openssl x509 -in cert.pem -noout -modulus

# Get modulus from private key
openssl rsa -in key.pem -noout -modulus
```
If both outputs are identical → the key and cert match.

---

## What Is Inside a CSR (Certificate Signing Request)

Three sections, similar structure to a certificate:

```
[ Certificate Request Info ]  ← the important stuff
[ Signature Algorithm      ]  ← hashing algo + key type
[ Signature                ]  ← signed by the SERVER's private key
```

### Fields Inside Certificate Request Info

- **Version** — always `0x00` (CSR v1 is the only version)
- **Subject DN** — filled in by the server, becomes the Subject of the final certificate
- **Public Key** — the server's public key (modulus + exponent for RSA)
- **Attributes** — optional fields:
  - **ExtensionsRequest** — server requests specific extensions for the cert (e.g. SAN entries)
  - **Challenge Password** — legacy mechanism for authorizing revocation, ignored by modern CAs

---

## Certificate and Key File Formats

### DER
- **Binary encoded** (raw ones and zeros)
- Format used on the wire during TLS
- Cannot be opened in a text editor
- Extensions: `.der`, `.cer`, `.crt`

### PEM
- **Base64 encoded** version of DER
- Most common and easiest to work with
- Can be opened in any text editor
- Identified by headers like `-----BEGIN CERTIFICATE-----` or `-----BEGIN PRIVATE KEY-----`
- One PEM file can contain multiple entities (e.g. cert + chain certs + key)
- Extensions: `.pem`, `.crt`, `.cer`, `.key`

### PFX / PKCS#12
- **Binary encoded**
- Contains: certificate + matching private key + optional chain certs — **all in one file**
- Originally PFX (Microsoft 1996), standardized as PKCS#12 (RSA Labs 1999)
- Easy to deploy because everything is bundled together
- Extensions: `.pfx`, `.p12`

### PKCS#7
- **Base64 encoded**
- Contains: certificates and/or chain certs — **no private keys ever**
- Originally designed for S/MIME email signing
- Identified by `-----BEGIN PKCS7-----`
- Extensions: `.p7b`, `.p7c`

### Format Quick Reference

| Format | Encoded | Contains Key? | Contains Cert? | Can Open in Text Editor? |
|---|---|---|---|---|
| DER | Binary | ✅ | ✅ | ❌ |
| PEM | Base64 | ✅ | ✅ | ✅ |
| PFX/PKCS#12 | Binary | ✅ | ✅ | ❌ |
| PKCS#7 | Base64 | ❌ | ✅ | ✅ |

---

## Useful OpenSSL Commands

```bash
# Connect to a server and grab its certificate
openssl s_client -connect reddit.com:443

# View certificate contents as human-readable text
openssl x509 -in cert.pem -text -noout

# View only the modulus of a certificate
openssl x509 -in cert.pem -noout -modulus

# View private key contents as human-readable text
openssl rsa -in key.pem -text -noout

# View only the modulus of a private key
openssl rsa -in key.pem -noout -modulus

# View CSR contents
openssl req -in csr.pem -text -noout
```

---

## Key Takeaways

- A **certificate** = server identity document, signed by a CA's private key
- A **private key** = just a file with math values — never leaves the server
- A **CSR** = request form sent to the CA, signed by the server's private key
- The **modulus** is the fingerprint linking a cert to its private key
- **X.509 v3 extensions** are what allow modern certs to support SANs, wildcards, constraints etc.
- **PEM** is the most human-friendly format; **PFX** is the most portable (cert + key bundled)
