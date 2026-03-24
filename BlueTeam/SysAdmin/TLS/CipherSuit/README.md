# Cipher Suites

## What Is a Cipher Suite

For a client and server to communicate securely, they need four things:

| Need | Purpose |
|---|---|
| **Key Exchange** | Generate a shared secret (seed value) over an unsecured medium |
| **Authentication** | Verify the server is who it claims to be |
| **Symmetric Encryption** | Protect bulk data transfer (confidentiality) |
| **Hashing (MAC)** | Verify data integrity and authenticity |

A **cipher suite** is a string that defines exactly which protocol is used for each of these four functions.

### Reading a Cipher Suite

```
TLS_DHE_RSA_WITH_AES_256_CBC_SHA
     ───  ───      ───────  ───
      │    │           │     │
      │    │           │     └── Hashing algorithm (SHA)
      │    │           └──────── Symmetric encryption (AES-256-CBC)
      │    └──────────────────── Authentication (RSA)
      └───────────────────────── Key exchange (DHE)
```

Another example where one protocol handles two roles:
```
TLS_RSA_WITH_RC4_128_MD5
    ───      ───────  ───
     │           │     │
     │           │     └── Hashing (MD5)
     │           └──────── Symmetric encryption (RC4-128)
     └──────────────────── Key exchange AND Authentication (RSA)
```

Cipher suites are not invented on the fly ,  they are chosen from a pre-built list maintained by **IANA**.

> Note: TLS 1.3 changes how cipher suites work significantly

---

## Key Exchange Protocols

The goal of key exchange is to establish a **seed value** ,  a shared secret number used to derive all symmetric session keys. This only happens once per connection.

### PSK (Pre-Shared Key)
- The seed value is **manually configured** by an admin on both sides before the handshake
- No actual exchange happens on the wire
- **Least secure** key exchange ,  but zero CPU cost
- Rare on the web; making a comeback with **IoT devices** (smart lights, speakers, etc.) that lack CPU power

### RSA Key Exchange
- Client **randomly generates** the seed value
- Client encrypts it with the server's **public key** and sends it across the wire
- Server decrypts it with its **private key** to recover the seed value
- **Problems:**
  - Seed value is only protected by the private key ,  if the private key is ever compromised, all past sessions can be decrypted
  - Security depends entirely on one party's random number generator being unpredictable
  - **Does NOT provide forward secrecy**

### Diffie-Hellman Variants

All four remaining protocols are variants of Diffie-Hellman. DH allows two parties to establish a shared secret over an unsecured medium. Both parties contribute a random number, so security doesn't depend on a single entity's RNG.

| Protocol | Description |
|---|---|
| **DH** | Standard Diffie-Hellman, static parameters baked into cert/key ,  rare |
| **DHE** | Ephemeral DH ,  parameters are temporary, deleted after use |
| **ECDH** | Elliptic Curve DH, static parameters ,  more efficient than DH, also rare |
| **ECDHE** | Ephemeral ECDH ,  most secure, most recommended ✅ |

#### Why Ephemeral (E) Matters ,  Forward Secrecy

- **Non-ephemeral** (DH, ECDH): The Diffie-Hellman starting parameters are permanently written into the cert and key file. If that key is ever stolen, past sessions can be decrypted.
- **Ephemeral** (DHE, ECDHE): After the seed value is established, all starting parameters are **permanently deleted**. There is no record of them anywhere.

This property is called **forward secrecy** (also called Perfect Forward Secrecy, PFS):

> **Once encrypted, always encrypted** ,  even if the private key is stolen in the future, past sessions cannot be decrypted because the values used to generate those session keys no longer exist.

**TLS 1.3 requires forward secrecy for all cipher suites.**

#### EC (Elliptic Curve) vs Non-EC
- ECDH/ECDHE are **more secure and more efficient** than DH/DHE for the same key size

---

## Authentication Protocols

The goal is to verify that the server actually owns the private key matching the public key in its certificate.

### PSK Authentication
- Client and server prove they have the same pre-shared key
- No certificates needed ,  very lightweight
- Rare on the web, possible use with IoT

### RSA Authentication
- Server proves ownership of private key via digital signature
- Widely supported everywhere
- Key sizes must be **≥ 2048 bits** to be considered secure today

### DSA / DSS (Digital Signature Algorithm / Standard)
- DSS = the standard that mandates use of DSA
- Equally secure to RSA at equal key sizes
- **Problems:**
  - Historically limited to 1024-bit keys (some implementations still enforce this)
  - DSA math requires unique random numbers per signature ,  if a random number is reused, **the private key can be extracted** (catastrophic failure)
  - An RFC exists to generate random numbers deterministically, but not all implementations use it
- **Verdict: prefer RSA over DSA**

### ECDSA (Elliptic Curve DSA)
- More efficient than RSA ,  smaller keys achieve equal security

| RSA Key Size | Equivalent ECDSA Key Size |
|---|---|
| 1024-bit | 160-bit |
| 2048-bit | 224-bit |
| 3072-bit | 256-bit |

- Keys are smaller AND scale more efficiently
- Smaller certs → less bandwidth
- **Verdict: prefer ECDSA over RSA** ,  but you can support both simultaneously
  - The client sends supported cipher suites before the server sends its cert
  - Server can choose to send ECDSA cert or RSA cert based on what the client supports

---

## Symmetric Encryption Protocols

Used to protect bulk data transfer after the key exchange. Two types: **stream ciphers** and **block ciphers**.

### Stream Ciphers
- Encrypt plaintext **bit by bit** as it flows through
- Faster in software
- Better on devices without hardware AES chips (mobile, IoT, embedded)
- **Vulnerability:** susceptible to bit-reordering attacks ,  must be paired with a MAC

### Block Ciphers
- Break plaintext into fixed-size **blocks**, encrypt each block with a symmetric key
- Faster on PCs/servers/laptops that have **hardware AES chips**
- **Vulnerability:** without a proper operation mode, patterns in plaintext leak into ciphertext (no diffusion)

#### Why Block Ciphers Need Operation Modes

Default mode is **ECB (Electronic Code Book)** ,  encrypts each block independently with the same key. Identical plaintext blocks produce identical ciphertext blocks. **This is completely insecure.** (The encrypted penguin image still looks like a penguin.)

Two secure operation modes:

**CBC (Cipher Block Chaining)**
- Each plaintext block is XOR'd with the **previous ciphertext block** before encryption
- First block uses an **Initialization Vector (IV)** ,  a random starting value generated alongside session keys
- Identical plaintext blocks → different ciphertext ✅
- **Problems:**
  - Last block gets **padded** to fill the block size → vulnerable to **padding oracle attacks**
  - Cannot be **parallelized** ,  must process block by block in order

**CTR (Counter Mode)**
- Each plaintext block is combined with a **unique nonce** (number used once)
- Nonce = random base value + incrementing counter per block
- Identical plaintext blocks → different ciphertext ✅
- **Problems:**
  - Ciphertext blocks are independent ,  vulnerable to **block reordering** attacks → must be paired with a MAC
- **Benefits:**
  - Can be **fully parallelized** ,  great for multi-core systems

**GCM (Galois Counter Mode)**
- CTR mode + **built-in MAC** combined into one step
- This makes it an **AEAD cipher** (Authenticated Encryption with Associated Data)
- Does symmetric encryption AND message authentication in a single step
- No padding oracle vulnerabilities
- **AEAD ciphers are the future** ,  required in TLS 1.3

---

### Protocol Comparison

| Protocol | Key Size | Type | Status |
|---|---|---|---|
| DES | 56-bit | Block | ❌ Insecure ,  brute-forceable in hours |
| RC4-128 | 128-bit | Stream | ❌ Mathematically broken |
| 3DES (Triple DES) | 112-bit effective | Block | ❌ Below 128-bit threshold ,  insecure |
| AES-128-CBC | 128-bit | Block | ✅ Acceptable |
| AES-256-CBC | 256-bit | Block | ✅ Secure (padding oracle risk mitigated in TLS 1.1+) |
| AES-128-GCM | 128-bit | Block (AEAD) | ✅✅ Preferred |
| AES-256-GCM | 256-bit | Block (AEAD) | ✅✅ Preferred |
| ChaCha20 | 256-bit | Stream (AEAD) | ✅✅ Preferred |

#### AES-128 vs AES-256
- Both are secure ,  the difference is debated
- AES-256 uses significantly more CPU for marginal security gain
- Analogy: one deadbolt vs ten deadbolts ,  one is already secure enough for most use cases

#### ChaCha20
- Stream cipher, always paired with **Poly1305** MAC → becomes an AEAD cipher
- Faster than AES in **software** (no hardware acceleration required)
- Better for mobile, IoT, embedded systems
- Good fallback if AES is ever compromised
- Named after a variant of Salsa20 (the creator just liked the name)

---

## Hashing Algorithms (Used as MAC)

All hashing algorithms in cipher suites are used within a **MAC (Message Authentication Code)** to provide data integrity and authentication.

**How HMAC works:**
1. Combine message + padding + secret key → digest 1
2. Combine digest 1 + more padding + secret key → digest 2 (the final MAC)

> The key insight: HMAC is more secure than direct hashing because the secret key is never visible and intermediate digests are hidden.

**General rule: larger digest = more secure (fewer collision chances)**

| Algorithm | Digest Size | Status |
|---|---|---|
| MD5 | 128-bit | ❌ Insecure for direct hashing (since 2010). Still okay in HMAC but avoid it. |
| SHA-1 | 160-bit | ❌ Insecure for direct hashing/signatures. Still okay in HMAC but avoid it. |
| SHA-256 | 256-bit | ✅ Secure for hashing, signatures, HMAC |
| SHA-384 | 384-bit (truncated 512) | ✅ Secure |
| Poly1305 | 128-bit | ✅ Faster than SHA-2, equally secure, always paired with ChaCha20 as AEAD |

> **Important:** SHA-1 is no longer acceptable for **certificate signatures** because creating a signature requires direct hashing ,  which is what's considered insecure.

---

## Avoid / Accept / Prefer Summary

### Key Exchange

| Avoid | Accept | Prefer |
|---|---|---|
| PSK | DHE (≥2048-bit keys) | **ECDHE** ✅ |
| RSA | RSA (≥2048-bit, only if SSL inspection required) | |
| DH | | |
| ECDH | | |

> Avoid anything without forward secrecy. Also avoid **null key exchange** ciphers entirely.

### Authentication

| Avoid | Accept | Prefer |
|---|---|---|
| PSK | RSA (≥2048-bit keys) | **ECDSA** ✅ |
| DSA/DSS | | Both RSA + ECDSA simultaneously |
| RSA <2048-bit | | |
| Export ciphers (40-bit keys) | | |

> **Export ciphers** were a legal artifact from US export law pre-1996. They limit keys to 40 bits. Formally deprecated in TLS 1.1 ,  avoid completely.

### Symmetric Encryption

| Avoid | Accept | Prefer |
|---|---|---|
| DES | AES-128-CBC (TLS 1.1+) | **AES-128-GCM** ✅ |
| RC4-128 | AES-256-CBC (TLS 1.1+) | **AES-256-GCM** ✅ |
| 3DES | | **ChaCha20** ✅ |
| Null encryption | | |

> Prefer AEAD ciphers (GCM, ChaCha20). Avoid null encryption ciphers in production.

### Hashing

| Avoid | Accept | Prefer |
|---|---|---|
| MD5 | SHA-1 (in HMAC only) | **SHA-256** ✅ |
| | | **SHA-384** ✅ |
| | | **Poly1305** ✅ |

> If choosing between SHA-2 and Poly1305, pick whichever is part of an AEAD cipher (paired with AES-GCM or ChaCha20).

---

## The Ideal Modern Cipher Suite

The most secure cipher suite you should prefer looks like this:

```
TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
      ────  ─────      ───────────  ─────
       │      │              │        │
       │      │              │        └── SHA-384 (hashing)
       │      │              └─────────── AES-256-GCM (AEAD encryption)
       │      └────────────────────────── ECDSA (authentication)
       └───────────────────────────────── ECDHE (key exchange, forward secrecy)
```

Or with ChaCha20:
```
TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
```

Both provide: forward secrecy + efficient keys + AEAD encryption + strong hashing.

---

## Key Takeaways

- A cipher suite defines **four protocols**: key exchange, authentication, encryption, hashing
- **Forward secrecy** (ECDHE/DHE) means past sessions can never be decrypted even if the private key leaks later
- **AEAD ciphers** (GCM, ChaCha20+Poly1305) do encryption and MAC in one step ,  the future of TLS
- Prefer **ECDSA** over RSA for authentication (smaller, more efficient keys)
- Prefer **ECDHE** for key exchange (forward secrecy + efficiency)
- **TLS 1.3** mandates forward secrecy and AEAD ciphers for all connections
- You can support both RSA and ECDSA simultaneously ,  server chooses based on client capabilities
