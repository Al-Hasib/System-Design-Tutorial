# Security Fundamentals: TLS, Encryption, AuthN/AuthZ & Firewalls

**Difficulty:** Intermediate/Advanced
**Estimated length:** 16-20 min
**Prerequisites:** [06 - HTTP/HTTPS & REST APIs Explained](../../Module-02-Networking-and-Communication/06-http-https-and-rest-apis/README.md), [09 - API Gateway & Backend-for-Frontend Pattern](../../Module-02-Networking-and-Communication/09-api-gateway-and-bff-pattern/README.md), [31 - Microservices Communication and Service Discovery](../../Module-07-Architecture-Patterns/31-microservices-communication-and-service-discovery/README.md)

## Learning Objectives

- Distinguish encryption from hashing and explain when each is used.
- Explain symmetric vs. asymmetric encryption and how TLS combines both.
- Clearly distinguish authentication from authorization, and describe common mechanisms for each (sessions, JWTs, OAuth2/OIDC).
- Explain mTLS and how service meshes use it to secure service-to-service traffic.
- Describe the role of firewalls, network segmentation, and a Web Application Firewall (WAF) in a defense-in-depth strategy.
- Recognize, at a system-design level, why defense-in-depth (multiple independent layers of security) matters more than any single control.

## Script

### Hook / Intro

Every topic in this course so far has quietly assumed the network is a safe place to send data. It isn't. Anyone on the network path — a compromised router, a malicious Wi-Fi hotspot, an attacker who's breached one service in your fleet — can potentially read or tamper with traffic, impersonate a user, or move laterally from one system to the next. Security isn't a separate specialty bolted onto system design after the fact; it's a first-class design constraint, exactly like scalability or availability. Today we cover the fundamentals every backend engineer needs: encryption, the crucial difference between authentication and authorization, and the network-level defenses that keep a system safe even when one layer fails.

### Encryption vs. Hashing

These two get confused constantly, so let's separate them cleanly. **Encryption** is reversible: data is transformed using a key such that anyone with the right key can decrypt it back to the original — used whenever you need to *recover* the original data later (data in transit, data at rest on disk). **Hashing** is one-way: it produces a fixed-size fingerprint of data that you cannot reverse back to the original — used when you only ever need to *verify* something matches, never recover it, which is exactly why passwords should be hashed (with a slow, salted algorithm like bcrypt or Argon2), never encrypted — if your database leaks, you want it to be computationally infeasible to recover the original passwords at all, not just inconvenient.

Within encryption, there are two families. **Symmetric encryption** (AES is the standard today) uses one shared secret key for both encrypting and decrypting — it's fast, but both sides need to already have that same key, which creates a chicken-and-egg problem: how do you safely share the key in the first place? **Asymmetric encryption** (RSA, elliptic-curve cryptography) uses a mathematically-linked key pair: a public key anyone can have, and a private key only the owner holds. Data encrypted with the public key can only be decrypted with the private key. It's much slower than symmetric encryption, but it solves the key-distribution problem — you can publish your public key openly.

### TLS: Combining Both

This is exactly why TLS (which we touched on back in video 6 for HTTPS) uses both. During the TLS handshake, asymmetric encryption is used just long enough to authenticate the server (via its certificate) and safely agree on a fresh, random symmetric key — solving the key-distribution problem using the slow-but-safe method. Then every byte of the actual session uses fast symmetric encryption with that negotiated key. You get asymmetric crypto's key-distribution safety and symmetric crypto's speed, without paying the cost of asymmetric encryption for the entire, potentially large, data transfer.

### Authentication vs. Authorization

If you remember one distinction from this video, make it this one, because interviewers ask it constantly and production incidents happen because engineers conflate them: **Authentication (AuthN)** answers "who are you?" — verifying identity. **Authorization (AuthZ)** answers "what are you allowed to do?" — checking permissions, and it only makes sense *after* authentication has already established who's asking. This maps directly onto HTTP status codes from video 6: 401 Unauthorized really means "authentication failed, I don't know who you are"; 403 Forbidden means "I know who you are, and you're not allowed to do this."

Common authentication mechanisms: **session-based auth**, where the server issues an opaque session ID stored in a cookie and keeps the actual session state server-side (or in a shared store like Redis); **JWT (JSON Web Tokens)**, self-contained signed tokens that carry identity claims directly in the token itself, letting any service verify the signature without a round trip to a central session store — useful in stateless, horizontally-scaled architectures, at the cost of harder immediate revocation since the token is valid until it expires; and **OAuth 2.0 / OpenID Connect**, the standard for delegated authorization and identity — OAuth2 lets a user grant a third-party app limited access to their data without sharing their password, and OIDC layers standardized identity (who is this user) on top of OAuth2's authorization flow. This is the "Sign in with Google" pattern you've used a hundred times.

### Securing Service-to-Service Communication

Everything above covers a client authenticating to your system. But inside a microservices architecture (recall Module 7), services also need to trust each other. **mTLS (mutual TLS)** extends the TLS handshake so *both* sides present certificates — the client proves its identity to the server, and the server proves its identity to the client, rather than only the server authenticating as in ordinary HTTPS. This is exactly the mechanism service meshes like Istio and Linkerd (mentioned briefly back in video 31) automate via sidecar proxies: every service-to-service call is automatically wrapped in mTLS, so even if an attacker gets a foothold inside your network, they can't simply impersonate another service or eavesdrop on internal traffic — every internal call still requires a valid, verified certificate.

### Firewalls & Network Segmentation

At the network layer, a **firewall** controls what traffic is allowed to reach a given resource based on rules — source/destination IP, port, protocol. In cloud environments this usually takes the form of security groups or network ACLs, which let you say "only the load balancer can reach the application servers on port 443; only the application servers can reach the database on port 5432" — nothing else gets through, even if it's on the same network. This is **network segmentation**: dividing your infrastructure into zones with different trust levels, so a compromise in one zone (say, a public-facing web tier) doesn't automatically grant access to a more sensitive zone (like your database or internal admin tools). A **Web Application Firewall (WAF)** operates one layer up, inspecting HTTP traffic itself for known attack patterns — SQL injection attempts, cross-site scripting payloads — and blocking them before they ever reach your application code, complementing rather than replacing your application's own input validation.

### Defense in Depth

The unifying principle behind everything in this video is **defense in depth**: no single control is assumed to be perfect, so you layer multiple independent defenses such that one failing doesn't mean total compromise. TLS encrypts traffic in transit, but you still hash passwords at rest in case the database itself is ever exposed. A firewall restricts what can reach your database, but you still require authentication and authorization on every request in case the firewall rule is ever misconfigured. mTLS secures internal service traffic, but you still validate and authorize every request at the service level rather than trusting anything that made it past the network perimeter — an approach often called "zero trust." None of these layers is optional because another one exists; each is there specifically for the scenario where a different layer fails.

### Real-World Example

Consider a payments feature end-to-end. The mobile app connects over HTTPS (TLS, from video 6) to your API gateway, which validates a JWT to authenticate the user and checks their authorization before allowing the "charge card" action — 401 if the token's invalid, 403 if they're authenticated but not allowed to charge this particular account. The gateway forwards the request to a payments microservice over mTLS, so the payments service can verify this call genuinely came from the trusted gateway and not an attacker who breached some unrelated internal service. A security group ensures only the payments service — nothing else in the cluster — can reach the database holding tokenized card data, and that data is encrypted at rest. And a WAF in front of the gateway blocks obvious injection attempts before they ever reach any of this. Every layer assumes the others might fail, and none of them alone is "the" security control.

### Recap

Encryption is reversible (used for confidentiality), hashing is one-way (used for verification, like passwords). TLS combines slow-but-key-distributable asymmetric encryption with fast symmetric encryption. Authentication establishes who you are; authorization establishes what you're allowed to do — they're different questions, mapped to 401 versus 403. mTLS extends TLS so services mutually authenticate each other, which is what service meshes automate. Firewalls and network segmentation restrict what can reach what at the network level, and a WAF blocks known attack patterns at the HTTP layer. None of it works in isolation — defense in depth means every layer assumes the others could fail.

### What's Next

That closes out Module 8 — we've now gone from the transport layer all the way up through message formats and security. From here, Module 9 goes deeper still on two things we've so far only covered at a conceptual level: how a database actually enforces isolation between concurrent transactions, and how its storage engine physically persists data on disk — plus a look at GraphQL as a genuine alternative to REST.

## Key Takeaways

- Encryption is reversible (for confidentiality); hashing is one-way (for verification) — always hash passwords, never merely encrypt them.
- Symmetric encryption is fast but needs a pre-shared key; asymmetric encryption solves key distribution but is slow — TLS uses asymmetric crypto briefly to exchange a symmetric key, then encrypts the session with that fast symmetric key.
- Authentication (who are you?) and authorization (what can you do?) are distinct steps — mapped to HTTP 401 vs. 403 respectively — and authorization only makes sense after authentication.
- Sessions, JWTs, and OAuth2/OIDC are the standard mechanisms for authenticating clients and delegating access; mTLS is how services authenticate each other, commonly automated by a service mesh.
- Firewalls/security groups enforce network segmentation, and a WAF blocks known HTTP-layer attacks — but defense in depth means no single layer is trusted alone; every layer assumes another might fail.
