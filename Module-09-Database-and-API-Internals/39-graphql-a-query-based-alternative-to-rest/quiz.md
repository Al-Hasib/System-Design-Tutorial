# Practice & Interview Questions

**1. Define over-fetching and under-fetching, and explain why they're structural problems with REST's fixed-shape endpoints.**
Over-fetching is receiving more fields than the client needs, because a REST endpoint returns the same fixed shape to every caller regardless of what they actually need. Under-fetching is not receiving enough combined data from one endpoint, forcing multiple round trips or a bespoke aggregating endpoint. Both stem from REST endpoints having one fixed response shape rather than letting the client specify what it wants.

**2. How does GraphQL solve both over-fetching and under-fetching at once?**
The client sends a single query specifying exactly which fields it wants, nested as deeply as needed, from one endpoint. The server returns a response shaped exactly like the query — nothing unused (no over-fetching) and everything needed in one round trip (no under-fetching).

**3. What is the N+1 query problem in GraphQL, and give a concrete example.**
It's when resolving a list of items and a related field on each triggers one query for the list plus one additional query per item. Example: fetching 20 users (1 query) then naively fetching each user's orders individually (20 more queries) = 21 total queries instead of 2.

**4. How does a DataLoader/batching pattern fix the N+1 problem?**
Instead of firing a database query the instant each resolver runs, it collects all requests for the same type of data across one query execution and issues them as a single batched call — e.g., one "fetch orders for these 20 user IDs" query instead of 20 separate "fetch orders for user X" queries.

**5. Why is HTTP-level caching (CDNs, browser cache, `Cache-Control`) much harder to apply to GraphQL than to REST?**
REST's `GET` requests map naturally onto HTTP caching because each distinct resource has its own URL that caches can key on. GraphQL typically uses a single `POST` endpoint for all queries, which HTTP caching infrastructure can't distinguish or cache by URL — production GraphQL systems need their own caching layer (client-side normalized caches, persisted queries) instead.

**6. How does GraphQL's approach to API versioning differ from REST's typical approach?**
REST often versions via the URL (`/v1/`, `/v2/`) when a breaking change is needed, potentially running multiple versions simultaneously. GraphQL's convention is to avoid versioning by only ever adding new fields to the schema and deprecating (not removing) old ones, so existing queries keep working as the schema evolves — though a truly breaking change eventually still requires a harder decision.

**7. Why is rate limiting structurally harder for a GraphQL API than a REST API?**
A REST request to a specific endpoint represents bounded, predictable work. A single GraphQL query can nest arbitrarily deep and request large amounts of related data in one call, so simple per-request rate limiting isn't enough — production GraphQL APIs typically need query complexity analysis or depth-limiting to prevent one expensive query from consuming disproportionate backend resources.

**8. Scenario: A company has iOS, Android, and web teams, each needing slightly different combinations of data on frequently-changing screens, all served by one backend team. Would you recommend REST or GraphQL, and why?**
GraphQL — this is exactly the scenario it was designed for: multiple independently-evolving clients with different, changing data needs, sharing one backend schema. Frontend teams can adjust their queries to fetch new field combinations without requiring the backend team to build or maintain bespoke aggregating endpoints for each client and screen.

**9. Scenario: A company is building a simple, public, third-party-facing API for partners to integrate with, prioritizing easy caching and straightforward documentation. Would you recommend REST or GraphQL, and why?**
REST — the workload favors well-understood, cacheable, per-resource endpoints, and third-party integrators generally benefit from REST's simpler mental model and native HTTP caching over GraphQL's added flexibility, which brings caching and rate-limiting complexity that isn't needed here.

**10. True or False: A GraphQL schema is only useful to the server; clients have no way to introspect what fields are available.**
False. GraphQL schemas are introspectable — clients and tools (like GraphiQL or Apollo's tooling) can query the schema itself to power autocomplete, type-checking, and auto-generated documentation, which is one of GraphQL's practical advantages over REST APIs that often rely on separately hand-maintained documentation.
