# Practice & Interview Questions

**1. What is the key difference between encryption and hashing, and why should passwords be hashed rather than encrypted?**
Encryption is reversible — anyone with the key can recover the original data. Hashing is one-way and cannot be reversed. Passwords should be hashed (with a slow, salted algorithm like bcrypt or Argon2) so that even if the database leaks, recovering the original passwords is computationally infeasible — encrypting them would mean anyone who also obtained the decryption key could recover every password directly.

**2. Why does TLS use both asymmetric and symmetric encryption instead of just one?**
Asymmetric encryption solves the key-distribution problem (a public key can be shared openly) but is too slow to encrypt an entire session's worth of data. Symmetric encryption is fast but requires both sides to already share a secret key. TLS uses asymmetric crypto briefly during the handshake to safely agree on a random symmetric key, then uses fast symmetric encryption for the actual session data.

**3. Explain the difference between authentication and authorization, and how it maps to HTTP status codes.**
Authentication verifies who a party is; authorization verifies what that (already-identified) party is allowed to do. A 401 Unauthorized response means authentication failed or is missing (the server doesn't know who you are). A 403 Forbidden response means authentication succeeded but the authenticated party isn't permitted to perform the requested action.

**4. What trade-off does using JWTs for authentication introduce compared to server-side sessions?**
JWTs are self-contained and signed, so any service can verify them without a round trip to a central session store — good for stateless, horizontally-scaled systems. The trade-off is revocation: since the token itself is the proof of identity and is valid until it expires, immediately invalidating a specific token before its natural expiry is harder than with server-side sessions, where you can simply delete the session record.

**5. What is mTLS, and how does it differ from the TLS used in ordinary HTTPS browsing?**
In ordinary HTTPS/TLS, only the server presents a certificate to prove its identity to the client. mTLS (mutual TLS) requires both sides to present certificates — the client also proves its identity to the server — which is how services can cryptographically verify each other's identity for service-to-service communication.

**6. How does a service mesh like Istio or Linkerd typically implement mTLS across a microservices architecture?**
Via sidecar proxies deployed alongside each service instance, which automatically handle the mTLS handshake and certificate management for every service-to-service call, so individual services don't need to implement mTLS themselves — the mesh enforces mutual authentication and encryption transparently at the infrastructure layer.

**7. What's the difference between a firewall/security group and a Web Application Firewall (WAF)?**
A firewall or security group operates at the network layer, controlling which sources can reach which destinations based on IP, port, and protocol. A WAF operates at the application (HTTP) layer, inspecting request content itself for known attack patterns like SQL injection or cross-site scripting and blocking them before they reach application code.

**8. What is network segmentation, and why does it limit the blast radius of a breach?**
Network segmentation divides infrastructure into zones with different trust levels (e.g., public web tier, internal services, database tier), enforced by firewall/security group rules so that only specific, expected traffic can cross zone boundaries. If an attacker compromises one zone (say, a public-facing service), segmentation prevents them from directly reaching a more sensitive zone (like the database) without also breaching the rules governing that boundary.

**9. Explain "defense in depth" and why a system shouldn't rely on just one strong security control.**
Defense in depth means layering multiple independent security controls so that no single point of failure leads to total compromise. Every control can fail or be misconfigured — TLS doesn't help if the database itself is compromised and passwords weren't hashed; a firewall rule can be misconfigured, so authentication/authorization checks still matter even for "internal" traffic. Each layer is designed to catch what another layer might miss.

**10. Scenario: An attacker compromises one microservice inside your cluster's internal network. What two controls from this video would limit what they can do next, and how?**
mTLS/service mesh — the compromised service still can't impersonate other services or decrypt traffic between services it isn't a genuine party to, since every call requires a valid certificate. Network segmentation/security groups — the compromised service can likely still only reach the specific destinations its own security group rules allow, not the entire internal network, limiting lateral movement.

**11. What do OAuth 2.0 and OpenID Connect (OIDC) each solve, and how do they relate to each other?**
OAuth 2.0 solves delegated authorization — letting a user grant a third-party app limited access to their data without sharing their password. OIDC is an identity layer built on top of OAuth2 that standardizes how to also establish *who* the user is (authentication), which is why "Sign in with Google" flows use OIDC on top of OAuth2's authorization mechanics.

**12. True or False: Once traffic is inside your internal network (past the perimeter firewall), it's safe to skip authentication/authorization checks between services.**
False — this is exactly the assumption "zero trust" and defense-in-depth argue against. A perimeter firewall can be misconfigured or bypassed, and an attacker who compromises any single internal service should not automatically gain trusted access to every other service. Internal calls should still be authenticated (e.g., via mTLS) and authorized independently.
