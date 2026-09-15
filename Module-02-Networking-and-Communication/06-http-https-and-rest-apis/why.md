# Why This Topic Matters: HTTP/HTTPS & REST APIs

> **In one sentence:** HTTP is the contract that lets code written by strangers, in different languages, on different continents, interoperate — and REST is the set of conventions that keeps that contract from turning into 200 bespoke, undocumented rules.

## The World Before This Idea

Imagine every API invented its own rules. One service signals errors by returning `{"ok": false}` with HTTP 200. Another returns HTTP 500 for "user not found." One uses `POST /getUser`, another `GET /user/delete?id=5`. Caching is impossible because nothing indicates whether a response is safe to reuse. Retries are dangerous because nothing indicates whether an operation is safe to repeat.

This isn't hypothetical — it's what a large fraction of internal APIs actually look like. The cost is paid every day: every integration needs custom code, every client needs custom error handling, and generic infrastructure (proxies, CDNs, gateways, monitoring) can't help you because it can't understand your traffic.

## The Problems It Solves

### 1. Infrastructure that can't do its job
**What you see:** Your CDN caches nothing. Your load balancer can't retry safely. Your monitoring can't tell errors from successes.

**Why it happens:** Every piece of shared infrastructure on the request path makes decisions based on HTTP semantics — the method, the status code, the cache headers. If a write is sent as `GET` or an error returns `200`, the infrastructure is being lied to, and it behaves accordingly.

**How HTTP semantics solve it:** `GET` is safe and cacheable, so proxies and CDNs can cache it. `PUT` and `DELETE` are idempotent, so a load balancer or client can retry them after a timeout without duplicating work. `POST` is neither, so nothing retries it blindly. Status codes tell every intermediary, in one number, what happened. Following the semantics is what lets you get caching, retries, and monitoring *for free* instead of building them.

### 2. Retries that silently create duplicates
**What you see:** A user is charged three times. The client timed out and retried; all three requests actually succeeded server-side.

**Why it happens:** The operation was not idempotent, and nothing in the protocol said so. Timeouts are ambiguous — the client cannot distinguish "never arrived" from "succeeded but the response was lost."

**How this topic solves it:** Understanding which methods are idempotent, and how to make non-idempotent operations idempotent with an idempotency key, is what makes retries safe. In a distributed system retries are not optional, so this is not a detail.

### 3. API designs that need a meeting to use
**What you see:** Every new client integration takes a week and a Slack thread, because the API's shape has to be explained rather than inferred.

**Why it happens:** Without conventions, the API encodes one team's habits. Resource naming, error format, pagination, and filtering are all invented locally and differently each time.

**How REST solves it:** Resource-oriented URLs, standard methods, and standard status codes mean a competent engineer can guess most of your API correctly before reading the docs. That predictability is the actual value of REST — not architectural purity.

### 4. Credentials and data readable by anyone on the path
**What you see:** A session token captured on public Wi-Fi is replayed to impersonate a user.

**Why it happens:** Plain HTTP is transmitted in the clear. Every router, Wi-Fi access point, and ISP between the user and your server can read and modify it.

**How HTTPS solves it:** TLS provides encryption (nobody can read it), integrity (nobody can modify it undetected), and authentication (the client can verify it's really talking to your server, not an impostor). This is also why HTTPS is now a hard prerequisite for HTTP/2, service workers, geolocation, and most modern browser APIs.

## The Price You Pay

- **REST is a poor fit for some problems.** Highly relational client needs cause over-fetching and under-fetching (the problem GraphQL exists to solve). Real-time push doesn't fit request/response at all (hence WebSockets and SSE). High-throughput internal service-to-service calls pay real overhead in text headers and JSON parsing (hence gRPC).
- **Purity debates waste time.** Arguments about whether an endpoint is "truly RESTful" rarely improve a product. Consistency matters far more than orthodoxy.
- **HTTPS has costs.** Extra round trips for the handshake, CPU for encryption, and certificate lifecycle management. All are small and worth it, but they're not zero — which is why connection reuse and session resumption matter.

## When You Need It — and When You Don't

| REST over HTTP fits when | Reach for something else when |
|---|---|
| Public APIs, or anything third parties consume | You need server-push or bidirectional streams → WebSockets/SSE |
| CRUD-shaped resources map naturally | Clients need flexible, nested queries → GraphQL |
| You want caching and proxies to work for free | Internal, latency-critical, high-volume RPC → gRPC |
| Broad client compatibility matters | You're moving huge binary payloads |

## Why This Shows Up in Interviews

API design appears in almost every system design round, usually as "what does the API look like?" Interviewers watch for correct method choice, sensible status codes, idempotency awareness, pagination for list endpoints, and versioning. Getting `POST` versus `PUT` right and mentioning idempotency keys for a payments endpoint is a small thing that signals real production experience.

## How It Connects

HTTP semantics are what make several later topics possible. **Caching and CDNs** rely on `GET` being safe and on cache-control headers. **Load balancers and API gateways** route and retry based on methods, paths, and status codes. **Idempotency** is a direct extension of HTTP method semantics into distributed systems. **gRPC and GraphQL** are best understood as responses to specific places where REST over HTTP is a poor fit.

**Next:** [Load Balancing Explained](../07-load-balancing-explained/why.md) — how all these requests get spread across more than one server.
