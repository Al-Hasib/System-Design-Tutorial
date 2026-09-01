# GraphQL: A Query-Based Alternative to REST

**Difficulty:** Intermediate
**Estimated length:** 12-15 min
**Prerequisites:** [06 - HTTP/HTTPS & REST APIs Explained](../../Module-02-Networking-and-Communication/06-http-https-and-rest-apis/README.md), [09 - API Gateway & Backend-for-Frontend Pattern](../../Module-02-Networking-and-Communication/09-api-gateway-and-bff-pattern/README.md)

## Learning Objectives

- Explain the over-fetching and under-fetching problems that REST's fixed-shape endpoints structurally create.
- Describe how GraphQL's single endpoint and client-specified queries address both problems.
- Explain the N+1 query problem in GraphQL resolvers and how batching (DataLoader-style) solves it.
- Compare GraphQL and REST on caching, versioning, and tooling.
- Decide when GraphQL is worth its added complexity for a given system.

## Script

### Hook / Intro

Back in video 6, we built REST around a clean idea: resources, identified by URLs, manipulated with HTTP methods. That model works beautifully until your mobile app's home screen needs a user's name, their last three orders' totals, and their five most recent notifications — all in one screen load. Do you make three separate REST calls? Build one bespoke `/home-screen` endpoint that returns exactly that combination and nothing else, and now maintain it forever as one more custom endpoint? This exact pain point is what GraphQL was built to solve, and understanding precisely what problem it solves — not just "GraphQL is trendy" — is what lets you make a real trade-off decision instead of a fashion choice.

### The Problem: Over-Fetching and Under-Fetching

REST endpoints have a fixed shape: `GET /users/5` returns whatever fields the server decided that endpoint returns — every time, to every client, regardless of what that particular client actually needs. This creates two opposite problems. **Over-fetching**: a mobile app that only needs a user's name and avatar still receives the full user object — bio, settings, join date, everything — wasting bandwidth on a slow mobile connection for data nobody's going to use. **Under-fetching**: the reverse case from our hook — one screen needs data assembled from multiple resources (user, orders, notifications), and a single REST call can't give you all of it, so the client either makes several round trips (each with its own latency cost) or the backend team builds a one-off aggregating endpoint for this exact screen, which doesn't generalize to the next screen that needs a slightly different combination.

Recall the Backend-for-Frontend pattern from video 9 — one common REST-world fix is a BFF per client type, aggregating calls behind the scenes. That works, but it means writing and maintaining a new backend service (or endpoint) every time a client's data needs to shift even slightly.

### GraphQL's Approach: One Endpoint, Client-Specified Shape

GraphQL flips the model. Instead of many URLs each returning a fixed shape, there's typically **one single endpoint**, and the client sends a **query** describing exactly which fields it wants, nested as deeply as it needs, in one request:

```
query {
  user(id: 5) {
    name
    avatarUrl
    orders(limit: 3) { total, createdAt }
    notifications(limit: 5) { message, read }
  }
}
```

The server returns a JSON response shaped exactly like the query — nothing more, nothing less. Over-fetching disappears because the client only asked for `name` and `avatarUrl`, not the whole user object. Under-fetching disappears because the client can pull user, orders, and notifications together in one round trip, in one request, without a bespoke aggregating endpoint. This is enforced by a strongly-typed **schema** — every GraphQL API publishes a schema defining exactly which types, fields, and relationships exist, which is also what powers GraphQL's excellent tooling: autocomplete, in-editor validation, and auto-generated documentation, all derived directly from the schema rather than hand-maintained separately (a real, chronic problem with REST API docs drifting out of sync with the actual API).

### The Real Cost: The N+1 Problem

GraphQL's flexibility has a genuine, well-known cost on the server side. Each field in a query is resolved by a **resolver** function, and naively, resolving `orders` for a user means one database query, and if each order also needs its line items, that's potentially one additional query *per order* — the classic **N+1 query problem**: 1 query for the user, N queries for their N orders, and possibly more nested underneath. A REST endpoint hand-written for a specific screen wouldn't have this problem, because a human wrote one efficient, purpose-built database query for that exact endpoint. GraphQL's generality — resolving each field somewhat independently — makes this a structural risk that has to be actively engineered around, typically using a **batching/dataloader pattern**: instead of firing a database query the instant each resolver runs, requests for the same type of data across the whole query are collected and issued as a single batched database call (e.g., "fetch orders for these 20 users" instead of 20 separate "fetch orders for user X" calls).

### Other Real Trade-offs

Caching is genuinely harder with GraphQL. REST's `GET` requests map naturally onto HTTP caching (CDNs, browser caches, `Cache-Control` headers, all keyed by URL) — recall the caching strategies from Module 4. GraphQL typically uses a single `POST` endpoint for everything, which defeats URL-based HTTP caching entirely; production GraphQL systems need their own caching layer (often at the client, like Apollo Client's normalized cache, or via persisted queries) rather than getting HTTP caching for free. Versioning also works differently: REST APIs often version via the URL (`/v1/`, `/v2/`) when breaking changes are needed; GraphQL's convention is to avoid versioning altogether by only ever adding new fields (never removing or changing existing ones), deprecating old fields in the schema instead — which works well until a genuinely breaking change is unavoidable. And rate limiting is structurally harder: with REST, a request to `/users/5` is bounded, predictable work; with GraphQL, a single query can nest arbitrarily deep and request enormous amounts of data in one call, so production GraphQL APIs typically need query complexity analysis or depth-limiting on top of ordinary request-based rate limiting (from Module 6) to prevent one expensive query from acting like a mini denial-of-service.

### Real-World Example

Consider a social media app's profile screen: name, avatar, bio, follower count, and the user's last 10 posts with like counts. In REST, this is either three-to-four separate calls from the client, or a custom `/profile-screen/:id` endpoint that some backend team now owns and has to keep updating every time a designer tweaks what the screen shows. In GraphQL, the client's own query simply asks for exactly that combination — no backend change needed when the frontend team decides to also show "mutual friends" next sprint; they just add a field to their query, as long as that field already exists somewhere in the schema. This is precisely why GraphQL took off first at companies like Facebook (where it originated) and Netflix — many different client teams (iOS, Android, web, TV apps), each with slightly different data needs, evolving independently against one shared backend schema.

### Recap

REST's fixed-shape endpoints create over-fetching (extra unused data) and under-fetching (multiple round trips or bespoke aggregating endpoints) — problems GraphQL solves by letting the client specify exactly the shape of data it wants from one endpoint, backed by a strongly-typed schema. That flexibility isn't free: resolvers risk the N+1 query problem (solved with batching/dataloaders), HTTP-level caching mostly stops working and needs its own solution, and rate limiting needs query-complexity analysis rather than simple per-request counting. GraphQL earns its complexity specifically when you have multiple, independently-evolving clients with meaningfully different data needs against a shared backend — not as a default replacement for REST everywhere.

### What's Next

That closes out this module's look at database and API internals. Next module, we shift focus entirely to distributed coordination — how multiple nodes safely share a lock, agree on the order events happened in, and answer questions like "have I seen this element before?" cheaply at massive scale.

## Key Takeaways

- REST's fixed-shape endpoints cause over-fetching (unused data sent) and under-fetching (multiple round trips or bespoke aggregating endpoints for combined data needs).
- GraphQL exposes one endpoint and a strongly-typed schema; clients specify exactly the fields and nesting they want, and the response matches that shape exactly.
- The N+1 query problem is GraphQL's core server-side risk — naive resolvers can trigger one database query per item in a list — solved with batching/dataloader patterns.
- GraphQL loses REST's free HTTP-level caching (single POST endpoint) and needs its own caching layer, plus query-complexity/depth limiting instead of simple per-request rate limiting.
- GraphQL earns its complexity when multiple independently-evolving clients have meaningfully different, changing data needs against one shared backend — not as a blanket REST replacement.
