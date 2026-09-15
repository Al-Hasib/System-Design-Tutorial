# Why This Topic Matters: Security Fundamentals — TLS, Encryption, AuthN/AuthZ & Firewalls

> **In one sentence:** Security is the only non-functional requirement where the failure mode is not degraded service but a headline, a regulatory fine, and users whose data is permanently out of their control — and almost every breach traces back to one of a small number of fundamentals done wrong.

## The World Before This Idea

Consider a system built with no security design, only features:

- Traffic is plain HTTP, so anyone on the same Wi-Fi can read session cookies and impersonate users.
- Passwords are stored as MD5 hashes, so a database dump is cracked within hours.
- The API checks that you are *logged in* but not that the record you requested is *yours*, so changing `?id=1042` to `?id=1043` returns someone else's invoice.
- The database is publicly reachable with a default password, because it was easier during development.
- A JWT is trusted without verifying its signature, so anyone can forge one.

None of these are exotic attacks. They are the ordinary top of every breach report, year after year. The uncomfortable truth is that most real-world compromises do not involve novel cryptography — they involve fundamentals that nobody owned.

## The Problems It Solves

### 1. Data readable and modifiable in transit
**What you see:** Session hijacking on shared networks, injected content from a compromised router, credentials harvested by a proxy.

**Why it happens:** Unencrypted traffic passes through many intermediaries you do not control — Wi-Fi access points, ISPs, transit providers.

**How TLS solves it:** Three properties, and it is worth knowing they are distinct. **Encryption** means intermediaries cannot read it. **Integrity** means they cannot modify it undetected. **Authentication**, via the certificate chain, means the client can verify it is talking to your server rather than an impostor — this last one is what actually stops man-in-the-middle attacks, and it is the reason certificate validation must never be disabled "temporarily."

### 2. Confusing "who you are" with "what you may do"
**What you see:** An authenticated user can read, modify, or delete records belonging to other users by changing an identifier. This class of flaw — broken object-level authorization — is consistently among the most common serious vulnerabilities in real APIs.

**Why it happens:** Authentication is centralized and easy to get right (a middleware checks the token). Authorization is per-resource, per-endpoint, and easy to forget in exactly one handler out of two hundred.

**How separating AuthN from AuthZ solves it:** Naming them as different concerns makes the second one a checklist item on every endpoint. *Authentication* establishes identity once; *authorization* must be evaluated for every resource access, on the server, against the authenticated identity — never based on an ID the client supplied and never enforced only in the UI.

### 3. Credentials that survive a database breach
**What you see:** A leaked password table is cracked and the credentials are replayed against other services, where many users reused them.

**Why it happens:** Fast hashes (MD5, SHA-1, plain SHA-256) are designed for speed, which is exactly wrong for passwords — a GPU tries billions per second.

**How proper hashing solves it:** bcrypt, scrypt, or Argon2 are deliberately slow and memory-hard, with a per-user salt to defeat rainbow tables. This turns an instant mass crack into an infeasible one. It is a one-line library choice that changes the outcome of a breach entirely.

### 4. Everything reachable from everywhere
**What you see:** One compromised web server gives an attacker direct network access to the database, the cache, the internal admin panel, and the message broker.

**Why it happens:** A flat network with no segmentation. Once inside, an attacker moves laterally without resistance.

**How network security solves it:** Defense in depth. Firewalls and security groups restrict which components can talk to which, on which ports. Databases sit in private subnets with no public route. Admin interfaces require a VPN or bastion. The goal is not to be unbreachable — it is to ensure one compromise does not become total compromise.

## The Price You Pay

- **Security adds friction, and friction gets bypassed.** Controls that make legitimate work painful get worked around — credentials pasted into chat, MFA disabled "for the demo," a firewall rule opened and never closed. A control nobody follows is worse than none, because it creates false confidence.
- **Latency and cost.** TLS handshakes add round trips, encryption costs CPU, and authorization checks add database lookups on every request. All are manageable, none are free.
- **Operational burden.** Certificates expire (a genuinely common cause of outages), keys must be rotated, secrets must be stored somewhere safe, and dependencies need patching continuously.
- **Rolling your own cryptography is a guaranteed loss.** Custom auth schemes, hand-built token formats, and homemade encryption fail in ways that are not visible until exploited. Use vetted libraries and established standards.
- **Security theater is a real risk.** Complex password rotation policies, security questions, and unreviewed compliance checklists consume effort while doing little — and can crowd out the fundamentals that matter.

## When You Need It — and When You Don't

There is no "when you don't" for the fundamentals. TLS everywhere, proper password hashing, authorization on every resource access, secrets outside the codebase, and dependencies patched are baseline for anything with real users.

What *does* scale with context is depth:

| Invest further when | Baseline is enough when |
|---|---|
| You handle payments, health, or personal data | It is an internal tool on a private network |
| You are subject to GDPR, HIPAA, PCI-DSS, SOC 2 | The data has no confidentiality value |
| You are a high-value target | — |
| You expose a public API to untrusted clients | — |

## Why This Shows Up in Interviews

Security is rarely the main question and is frequently a deciding follow-up: "how do you authenticate requests?", "where do you store secrets?", "how do you make sure a user can only see their own data?" Candidates who volunteer it — TLS in transit and at rest, tokens with short expiry and refresh, authorization checked server-side per resource, rate limiting on auth endpoints, secrets in a managed store — stand out precisely because so many do not mention it at all. In designs involving payments or personal data, omitting security entirely reads as a gap in judgment rather than a gap in knowledge.

## How It Connects

TLS runs over the **transport layer** (topic 33) and is usually terminated at the **reverse proxy** or **API gateway** (topics 8, 9), which is also where **rate limiting** (topic 25) protects authentication endpoints from brute force. Token validation at the gateway is central to **microservices communication** (topic 31). **Multi-region** and **replication** designs (topics 47, 13) must account for where data is legally allowed to reside, and **observability** (topic 43) is what lets you detect an intrusion — while also being a place where secrets and personal data leak into logs if you are careless.

**Next:** [Transaction Isolation Levels & Concurrency Control](../../Module-09-Database-and-API-Internals/37-transaction-isolation-levels-and-concurrency-control/why.md) — back inside the database, at the level where correctness under concurrency is decided.
