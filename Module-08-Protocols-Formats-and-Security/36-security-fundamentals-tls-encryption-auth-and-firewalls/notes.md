# Study Notes: Security Fundamentals

## Definitions

- **Encryption:** Reversible transformation of data using a key, so only holders of the correct key can read it. Used for confidentiality.
- **Hashing:** One-way transformation producing a fixed-size fingerprint; cannot be reversed. Used for verification (e.g., password storage, integrity checks).
- **Symmetric encryption:** One shared secret key used for both encryption and decryption (e.g., AES). Fast, but requires safe key distribution.
- **Asymmetric encryption:** A public/private key pair; data encrypted with the public key is decrypted only with the private key (e.g., RSA, ECC). Slower, but solves key distribution.
- **Authentication (AuthN):** Verifying *who* a party is.
- **Authorization (AuthZ):** Verifying *what* an authenticated party is allowed to do.
- **mTLS (mutual TLS):** TLS extended so both client and server present certificates, authenticating each other (not just the server, as in ordinary HTTPS).
- **Firewall:** A network control that allows/blocks traffic based on rules (source/destination IP, port, protocol).
- **WAF (Web Application Firewall):** Inspects HTTP traffic for known attack patterns (SQLi, XSS, etc.) and blocks them at the application layer.
- **Defense in depth:** Layering multiple independent security controls so that one failing does not mean total compromise.

## Encryption vs. Hashing

| | Encryption | Hashing |
|---|---|---|
| Reversible? | Yes (with the key) | No |
| Purpose | Confidentiality (recover original later) | Verification/integrity (never need original back) |
| Typical use | Data in transit (TLS), data at rest | Password storage, checksums, digital signatures |
| Example algorithms | AES (symmetric), RSA/ECC (asymmetric) | bcrypt, Argon2 (passwords); SHA-256 (integrity) |

## Symmetric vs. Asymmetric Encryption

| | Symmetric | Asymmetric |
|---|---|---|
| Keys | One shared secret key | Public/private key pair |
| Speed | Fast | Slow (much more computation) |
| Key distribution problem | Yes — must share the key safely first | No — public key can be shared openly |
| Example algorithm | AES | RSA, ECC |
| Where used in TLS | Encrypting the actual session data | Authenticating the server & exchanging the symmetric key |

## AuthN vs. AuthZ

| | Authentication | Authorization |
|---|---|---|
| Question answered | Who are you? | What are you allowed to do? |
| Failure status code | 401 Unauthorized | 403 Forbidden |
| Depends on the other? | Happens first | Requires authentication to have already succeeded |
| Common mechanisms | Sessions, JWT, OAuth2/OIDC login | Roles/permissions checks, scopes, policy engines |

## Authentication Mechanisms

| Mechanism | How it works | Trade-off |
|---|---|---|
| Session-based | Server issues opaque session ID (cookie); state stored server-side (e.g., Redis) | Easy instant revocation; needs shared session store to scale horizontally |
| JWT | Self-contained signed token carries identity claims | No server-side lookup needed (stateless, scales easily); harder to revoke before expiry |
| OAuth 2.0 | Standard for delegated authorization (grant limited access without sharing password) | Industry standard for third-party access ("Sign in with X") |
| OpenID Connect (OIDC) | Identity layer on top of OAuth2 | Standardizes "who is this user," not just "what can this app access" |

## Network & Application Layer Defenses

| Control | Layer | Purpose |
|---|---|---|
| Firewall / Security Group | Network | Restrict which sources can reach which destinations (IP/port/protocol) |
| Network segmentation | Network | Divide infrastructure into trust zones so a breach in one doesn't grant access to all |
| mTLS / Service Mesh | Transport | Mutual authentication + encryption between internal services |
| WAF | Application (HTTP) | Blocks known attack patterns (SQL injection, XSS) before reaching app code |

## Key Numbers / Facts

- AES (symmetric) is typically orders of magnitude faster than RSA (asymmetric) for the same amount of data — why TLS uses asymmetric crypto only briefly during the handshake.
- bcrypt and Argon2 are deliberately slow (tunable work factor) specifically to make brute-forcing leaked password hashes computationally expensive.
- OAuth 2.0 was finalized as RFC 6749 in 2012; OpenID Connect layers on top of it.

## Summary

- Encrypt when you need the original back (confidentiality); hash when you only need to verify a match (never store plaintext passwords).
- TLS combines asymmetric encryption (key exchange/server authentication) with symmetric encryption (fast session encryption).
- Authentication (who) and authorization (what) are distinct and sequential — 401 vs. 403.
- mTLS/service meshes secure service-to-service trust; firewalls/segmentation and WAFs secure the network and HTTP layers.
- No single control is trusted alone — defense in depth assumes any one layer could fail.
