# Why This Topic Matters: Forward Proxy vs Reverse Proxy

> **In one sentence:** Both sit in the middle of a connection, but one works on behalf of the client and the other on behalf of the server — and confusing them leads to security controls placed where they protect nobody.

## The World Before This Idea

Without an intermediary, every client talks directly to every server. That sounds clean, and it makes several important things impossible:

- Your backend servers' IP addresses are public, so they're directly attackable.
- Every backend has to terminate its own TLS, manage its own certificates, enforce its own rate limits, and write its own access logs.
- A company has no way to enforce policy on what its employees' machines can reach.
- There is nowhere to put a shared cache, because there is no shared point on the path.

A proxy is simply a deliberate choke point on the connection — and a choke point is where you can *do* things.

## The Problems It Solves

### 1. Cross-cutting concerns duplicated in every service
**What you see:** TLS termination, gzip compression, request logging, rate limiting, and IP allowlisting are implemented separately in twelve services, slightly differently, and three of them are out of date.

**Why it happens:** With no shared entry point, every service must handle everything itself.

**How a reverse proxy solves it:** One place in front of everything handles TLS (with one certificate lifecycle), compression, logging, and basic filtering. Services get to be plain HTTP servers that do business logic. This is the single biggest reason Nginx, HAProxy, and Envoy are in nearly every production stack.

### 2. Backends directly exposed to the internet
**What you see:** A scan finds your application servers, and they're taking traffic from bots, scrapers, and exploit attempts directly.

**Why it happens:** If clients connect straight to backends, backends must be publicly routable.

**How a reverse proxy solves it:** Only the proxy is public. Backends live on a private network and accept connections only from it. This shrinks the attack surface to one hardened component and gives you a natural place for a WAF and DDoS filtering.

### 3. Every request hitting the application, even identical ones
**What you see:** The same logo, the same CSS bundle, and the same popular API response are computed and served by the application thousands of times per second.

**Why it happens:** With no shared intermediary, there's nowhere to keep a shared cached copy.

**How a proxy solves it:** A reverse proxy can cache responses and serve them without ever touching the application. (A forward proxy does the mirror-image version of this — caching on behalf of many clients so a popular resource is fetched once for the whole office.)

### 4. No control over what leaves the network
**What you see:** A compromised internal service starts exfiltrating data to an external host, and nothing notices or stops it.

**Why it happens:** Outbound traffic is unmediated.

**How a forward proxy solves it:** All outbound requests are funneled through a proxy that can allowlist destinations, log every request, and block the rest. This is also how corporate content filtering, egress auditing, and stable outbound IP addresses (needed when a partner allowlists *you*) are implemented.

## The Price You Pay

- **Another hop, another failure domain.** The proxy adds latency and becomes something that must itself be redundant and monitored.
- **Debugging gets harder.** Client IPs are replaced unless `X-Forwarded-For` is handled correctly — and mishandling it is a classic source of broken rate limiting, wrong geolocation, and IP-spoofing vulnerabilities.
- **Caching bugs are nasty.** A misconfigured cache key can serve one user's personalized response to another. Caching at a shared layer requires care with auth-dependent responses.
- **Configuration drift.** Proxy config becomes its own codebase, and a bad reload can take down everything at once.

## When You Need It — and When You Don't

| Reverse proxy when | Forward proxy when |
|---|---|
| You're operating servers and want a single public entry point | You're operating clients and need policy over outbound traffic |
| You want centralized TLS, caching, compression, logging | You need a stable egress IP for partner allowlisting |
| You want to hide and protect backends | You need auditing or filtering of what leaves the network |
| You need request routing across services | You want a shared cache for many clients |

## Why This Shows Up in Interviews

Interviewers ask this to check conceptual precision, because the two are superficially identical and candidates frequently blur them. The clean framing — *a forward proxy is deployed by the client side and represents clients to the internet; a reverse proxy is deployed by the server side and represents servers to clients* — answers it in one sentence. Follow-ups usually go to what a reverse proxy is good for and how it differs from a load balancer and an API gateway.

## How It Connects

The reverse proxy is the general category that **load balancers**, **API gateways**, and **CDN edge nodes** are all specializations of. It's the natural home for **TLS termination** (see the security topic), the **HTTP caching layer**, and the enforcement point for **rate limiting**. Understanding the category makes all of those feel like variations rather than separate inventions.

**Next:** [API Gateway & Backend-for-Frontend Pattern](../09-api-gateway-and-bff-pattern/why.md) — what happens when that entry point grows real intelligence.
