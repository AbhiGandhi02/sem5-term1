# 9. Network Security Basics

## Why Network Security?

Network security protects data during transmission from:
- **Eavesdropping** (passive attack) — intercepting data
- **Tampering** (active attack) — modifying data in transit
- **Impersonation** — pretending to be someone else
- **Denial of Service** — overwhelming a system to make it unavailable

---

## Types of Attacks

### Passive Attacks (Hard to detect)
- **Eavesdropping / Sniffing** — listening to network traffic
- **Traffic Analysis** — analyzing patterns even if data is encrypted
- Goal: Gain information **without altering** data

### Active Attacks (Can be detected)
- **Modification** — altering messages in transit
- **Masquerade / Spoofing** — pretending to be another entity
- **Replay Attack** — capturing and resending valid messages
- **Denial of Service (DoS/DDoS)** — flooding target to prevent legitimate use
- **Man-in-the-Middle (MITM)** — intercepting and altering communication between two parties

---

## Cryptography Fundamentals

### Key Terms
| Term | Definition |
|------|------------|
| **Plaintext** | Original readable message |
| **Ciphertext** | Encrypted (unreadable) message |
| **Encryption** | Plaintext → Ciphertext (using a key) |
| **Decryption** | Ciphertext → Plaintext (using a key) |
| **Key** | A value used in the encryption/decryption algorithm |
| **Cipher** | The algorithm used for encryption |

### Types of Cryptography

#### 1. Symmetric Key Cryptography (Secret Key)
```
        Same Key (K)
           │
Plaintext → [Encrypt] → Ciphertext → [Decrypt] → Plaintext
    Sender                                         Receiver
```

- **Same key** used for both encryption and decryption
- **Fast** — suitable for bulk data encryption
- **Problem**: Key distribution — how to securely share the key?
- **Algorithms**:
  - **DES (Data Encryption Standard)**: 56-bit key, 64-bit block — now considered insecure
  - **3DES (Triple DES)**: DES applied 3 times, 168-bit effective key
  - **AES (Advanced Encryption Standard)**: 128/192/256-bit keys — current standard
  - **RC4**: Stream cipher — used in older SSL/WEP
  - **Blowfish**: 32–448 bit keys, fast

#### 2. Asymmetric Key Cryptography (Public Key)
```
        Public Key (Kpub)           Private Key (Kpriv)
             │                           │
Plaintext → [Encrypt] → Ciphertext → [Decrypt] → Plaintext
    Sender                                         Receiver
```

- **Two different keys**: Public key (shared) + Private key (secret)
- **Public key encrypts**, **Private key decrypts** (or vice versa for signing)
- **Slower** than symmetric — used for key exchange and digital signatures
- **Solves** key distribution problem
- **Algorithms**:
  - **RSA**: Most widely used, based on factoring large prime numbers
  - **Diffie-Hellman**: Key exchange protocol (not encryption itself)
  - **Elliptic Curve Cryptography (ECC)**: Shorter keys, same security as RSA
  - **DSA**: Digital Signature Algorithm

### Symmetric vs Asymmetric

| Feature | Symmetric | Asymmetric |
|---------|-----------|------------|
| **Keys** | 1 (shared secret) | 2 (public + private) |
| **Speed** | Fast | Slow |
| **Key Distribution** | Difficult | Easy (public key is public) |
| **Use Case** | Bulk data encryption | Key exchange, digital signatures |
| **Key Length** | Shorter (128-256 bits) | Longer (2048-4096 bits) |
| **Algorithms** | AES, DES, 3DES | RSA, ECC, Diffie-Hellman |

### Hybrid Approach (Used in Practice)
```
1. Use asymmetric encryption to securely exchange a symmetric key
2. Use the symmetric key for actual data encryption
3. Example: TLS/SSL does exactly this
```

---

## Hashing

- Converts input of **any length** into a **fixed-length output** (hash/digest)
- **One-way function** — cannot reverse hash to get original data
- **Deterministic** — same input always produces same hash
- Used for **integrity verification**, password storage, digital signatures

### Properties of a Good Hash Function
1. **Pre-image resistance**: Given hash, cannot find original input
2. **Second pre-image resistance**: Given input, cannot find different input with same hash
3. **Collision resistance**: Cannot find two different inputs with same hash

### Common Hash Algorithms

| Algorithm | Output Size | Status |
|-----------|-------------|--------|
| **MD5** | 128 bits | Broken (collisions found), avoid |
| **SHA-1** | 160 bits | Deprecated, not recommended |
| **SHA-256** | 256 bits | Secure, widely used |
| **SHA-512** | 512 bits | Secure, used for high security |
| **SHA-3** | Variable | Newest standard |

---

## Digital Signatures

Provide **authentication**, **integrity**, and **non-repudiation**.

### How It Works
```
SIGNING (Sender):
  Message → [Hash] → Digest → [Encrypt with Sender's Private Key] → Digital Signature
  Send: Message + Digital Signature

VERIFICATION (Receiver):
  Digital Signature → [Decrypt with Sender's Public Key] → Digest₁
  Message → [Hash] → Digest₂
  Compare: Digest₁ == Digest₂ ? → Verified! : Tampered!
```

### Properties
- **Authentication**: Confirms identity of sender (only sender has private key)
- **Integrity**: Any modification to message changes the hash → verification fails
- **Non-repudiation**: Sender cannot deny sending (only they have the private key)

---

## Digital Certificates

- **Binds a public key to an identity** (person, organization, server)
- Issued by a **Certificate Authority (CA)** — trusted third party
- Uses **X.509 standard**

### Certificate Contents
| Field | Description |
|-------|-------------|
| Subject | Identity (domain name, organization) |
| Issuer | CA that issued the certificate |
| Public Key | Subject's public key |
| Serial Number | Unique identifier |
| Validity Period | Start and expiration dates |
| Signature | CA's digital signature (verifies authenticity) |

### Certificate Chain of Trust
```
Root CA (self-signed, pre-installed in browsers)
  └→ Intermediate CA (signed by Root CA)
      └→ Server Certificate (signed by Intermediate CA)
```

---

## SSL/TLS (Secure Sockets Layer / Transport Layer Security)

- Provides **encryption, authentication, integrity** for network communication
- **SSL** is deprecated → **TLS** is the current standard (TLS 1.2, TLS 1.3)
- Operates between **Application Layer and Transport Layer**
- Used by **HTTPS**, email, VPN

### TLS Handshake (Simplified)
```
Client                              Server
  |                                    |
  |---- Client Hello ---------------→ |   (supported cipher suites, TLS version)
  |                                    |
  |←--- Server Hello + Certificate ---|   (chosen cipher suite, server's cert)
  |                                    |
  |   [Client verifies certificate]    |
  |                                    |
  |---- Key Exchange ---------------→ |   (pre-master secret encrypted with server's public key)
  |                                    |
  |   [Both derive session key]        |
  |                                    |
  |---- Finished (encrypted) -------→ |
  |←--- Finished (encrypted) --------|
  |                                    |
  |==== Encrypted Communication ======|
```

1. **Client Hello**: Client sends supported TLS versions, cipher suites, random number
2. **Server Hello**: Server chooses cipher suite, sends certificate (with public key)
3. **Key Exchange**: Client generates pre-master secret, encrypts with server's public key
4. **Session Keys**: Both sides derive symmetric session keys from pre-master secret
5. **Encrypted Communication**: All further data encrypted with symmetric session key

---

## Firewalls

A **security system** that monitors and controls incoming/outgoing network traffic based on predetermined rules.

### Types of Firewalls

| Type | Layer | Description |
|------|-------|-------------|
| **Packet Filter** | 3-4 | Filters based on IP, port, protocol. Stateless. |
| **Stateful Inspection** | 3-4 | Tracks connection state. More secure than packet filter. |
| **Application Gateway (Proxy)** | 7 | Inspects application data. Deep packet inspection. |
| **Next-Gen Firewall (NGFW)** | 3-7 | Combines all above + IPS, malware detection. |

### Firewall Rules Example
```
Rule 1: ALLOW TCP from 192.168.1.0/24 to ANY port 80,443 (allow web browsing)
Rule 2: ALLOW TCP from ANY to 10.0.0.5 port 22 (allow SSH to server)
Rule 3: DENY ALL from ANY to ANY (default deny everything else)
```

---

## Common Security Protocols Summary

| Protocol | Purpose | Layer |
|----------|---------|-------|
| **IPSec** | Encrypts IP packets (VPN) | Network (3) |
| **SSL/TLS** | Encrypts application data | Between 4-7 |
| **SSH** | Secure remote access | Application (7) |
| **HTTPS** | Secure web browsing | Application (7) |
| **WPA2/WPA3** | Wi-Fi security | Data Link (2) |
| **Kerberos** | Authentication protocol | Application (7) |
| **PGP** | Email encryption | Application (7) |

---

## IPSec (Brief Overview)

- Security protocol suite for **securing IP communications**
- Operates at **Network Layer (Layer 3)**
- **Two modes**:
  - **Transport Mode**: Only payload encrypted, IP header unchanged (host-to-host)
  - **Tunnel Mode**: Entire IP packet encrypted, new IP header added (gateway-to-gateway, VPN)
- **Two protocols**:
  - **AH (Authentication Header)**: Integrity + Authentication (no encryption)
  - **ESP (Encapsulating Security Payload)**: Integrity + Authentication + **Encryption**

---

## Quick Revision Checklist

- [ ] Passive vs Active attacks (examples of each)
- [ ] Symmetric encryption: same key, AES/DES, fast, key distribution problem
- [ ] Asymmetric encryption: public/private key pair, RSA, slow, solves key distribution
- [ ] Hybrid approach: asymmetric for key exchange, symmetric for data
- [ ] Hashing: one-way, MD5 (broken), SHA-256 (secure), integrity check
- [ ] Digital Signatures: Hash + encrypt with private key → authentication + integrity + non-repudiation
- [ ] Digital Certificates: X.509, CA, chain of trust
- [ ] SSL/TLS handshake steps: Hello → Certificate → Key Exchange → Encrypted session
- [ ] Firewall types: Packet filter, Stateful, Application gateway, NGFW
- [ ] IPSec: Transport vs Tunnel mode, AH vs ESP
- [ ] Common security protocols and their layers
