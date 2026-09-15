# Why This Topic Matters: GraphQL — A Query-Based Alternative to REST

> **In one sentence:** REST endpoints are shaped by the backend team, and every client then either downloads far more than it needs or makes six requests to assemble one screen — GraphQL moves the shape of the response to the client that actually knows what it needs.

## The World Before This Idea

A mobile app renders a user profile screen: name, avatar, follower count, and the titles of the last three posts.

With REST it makes `GET /users/42`, which returns the full user object — fifty fields including a bio, settings, notification preferences, and timestamps — because that endpoint serves the web app, the admin panel, and three other screens. It then calls `GET /users/42/posts`, and for each post calls `GET /posts/{id}/comments` to get a count. Six requests, each a full round trip on a mobile network, and the payload is several times what the screen displays.

The mobile team asks for a trimmed endpoint. The backend team adds `GET /users/42/profile-summary`. Then the tablet team asks for a slightly different one. Then the watch app. Six months later there are fourteen near-duplicate endpoints, each with its own tests and its own maintenance, and every UI change still requires a backend deploy.

## The Problems It Solves

### 1. Over-fetching and under-fetching
**What you see:** Payloads full of unused fields, alongside screens that need several sequential requests to assemble.

**Why it happens:** A REST endpoint returns a fixed resource shape, chosen once, for all consumers. It can be right for one client and wrong for the rest.

**How GraphQL solves it:** The client sends a query describing exactly the fields and relationships it wants, and gets exactly that — one request, one response, nothing extra. For mobile clients on high-latency, metered connections, eliminating both the extra bytes and the extra round trips is a substantial, user-visible improvement.

### 2. Backend deploys required for frontend changes
**What you see:** Adding one field to a screen means a backend ticket, a backend deploy, and a wait.

**Why it happens:** Response shapes live in backend code.

**How GraphQL solves it:** If the field already exists in the schema, the frontend simply asks for it. Frontend teams iterate without backend coordination, which is often the single biggest organizational reason teams adopt it.

### 3. Versioning that never ends
**What you see:** `/v1`, `/v2`, `/v3` running simultaneously, with old versions kept alive indefinitely because mobile clients in the wild cannot be forced to upgrade.

**Why it happens:** Any change to a shared response shape is potentially breaking, so the only safe move is a new version.

**How GraphQL solves it:** Adding a field breaks nobody, because clients only receive what they asked for. Fields are deprecated rather than removed, and — because every query is explicit — you can measure exactly which clients still request a deprecated field before retiring it. That measurability is a real advantage over REST, where you cannot tell who reads which field.

### 4. Discovery and integration friction
**What you see:** New clients need a walkthrough and a stale wiki page before they can call the API.

**Why it happens:** REST APIs are conventionally structured but not self-describing unless someone maintains an OpenAPI spec.

**How GraphQL solves it:** The schema is strongly typed and introspectable, so tooling gives autocompletion, type checking, and generated client types for free. The contract cannot drift from the implementation, because it is the implementation.

## The Price You Pay

GraphQL solves real problems and creates a distinctive set of new ones:

- **HTTP caching stops working.** Every query is typically a `POST` to a single `/graphql` endpoint, so CDNs, reverse proxies, and browser caches — which key on URL and method — cannot help. You must build caching at the field or resolver level instead. For read-heavy public content, this is a genuinely large loss.
- **The N+1 problem is the default.** A query for 100 posts with their authors naively issues 1 + 100 database queries, because each field resolves independently. DataLoader-style batching fixes it, and you must apply it deliberately and everywhere — it is not automatic.
- **Clients can write expensive queries.** Deeply nested or wide queries can be pathologically costly, and on a public API this is a denial-of-service vector. Mitigations (query depth limits, complexity scoring, persisted queries, allowlists) are mandatory, not optional — and each adds machinery.
- **Rate limiting is harder.** "100 requests per minute" is meaningless when one request can be a thousand times more expensive than another. You need cost-based limiting.
- **Observability is harder.** Every request hits the same endpoint with a 200 status, so per-endpoint dashboards, error rates, and latency breakdowns all need GraphQL-aware instrumentation.
- **It is a substantial amount of infrastructure** for an API with one client and simple needs, where REST is less code and less to go wrong.

## When You Need It — and When You Don't

| Use GraphQL when | Use REST when |
|---|---|
| Many client types with genuinely different data needs | One client, or clients with uniform needs |
| The data is graph-shaped with deep relationships | Resources are flat and CRUD-shaped |
| Mobile clients need minimal payloads and few round trips | HTTP caching and CDN offload are important |
| Frontend teams need to iterate independently | The API is public and must be simple to consume |
| You are aggregating several backend services | File uploads and downloads dominate |

A common and sensible middle ground: GraphQL as the internal BFF layer for first-party apps, REST for the public and partner API.

## Why This Shows Up in Interviews

GraphQL appears as an alternative when discussing API design, especially for mobile-heavy or multi-client systems. The signal interviewers look for is balance: candidates who propose it only for its benefits are weaker than those who name the costs — particularly the loss of HTTP caching and the N+1 resolver problem, since those are the two that actually bite teams in production. Framing it as a client-driven BFF, rather than as a wholesale replacement for REST, usually reads as the most mature position.

## How It Connects

GraphQL is essentially a generalized **BFF** (topic 9) where the aggregation is specified by the client at query time rather than hardcoded by a backend team. It is an alternative to **REST over HTTP** (topic 6), and its caching problem is a direct consequence of abandoning the HTTP semantics that make **caching** (topic 17) and **CDNs** (topic 18) work. Resolver batching is the same N+1 problem that appears in **microservices communication** (topic 31), and query-cost limiting is a specialized form of **rate limiting** (topic 25).

**Next:** [Distributed Locking](../../Module-10-Distributed-Coordination-and-Scale-Techniques/40-distributed-locking-redlock-zookeeper-and-etcd/why.md) — coordinating exclusive access across machines.
