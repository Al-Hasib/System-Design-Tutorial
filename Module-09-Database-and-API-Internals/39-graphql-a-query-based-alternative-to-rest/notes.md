# Study Notes: GraphQL vs. REST

## Definitions

- **Over-fetching:** A response contains more fields than the client actually needs, wasting bandwidth.
- **Under-fetching:** A single endpoint doesn't return enough combined data, forcing multiple round trips or a bespoke aggregating endpoint.
- **GraphQL:** A query language and runtime for APIs where clients specify the exact shape of data they want from a single endpoint, backed by a strongly-typed schema.
- **Resolver:** A function responsible for fetching the data for one field in a GraphQL query.
- **N+1 query problem:** Naive resolvers triggering one query per parent record plus one query per related child record (1 + N), instead of a single batched query.
- **DataLoader / batching pattern:** Collecting all requests for the same type of data within one GraphQL query execution and issuing them as a single batched backend call.

## REST vs. GraphQL

| Aspect | REST | GraphQL |
|---|---|---|
| Endpoints | Many, one per resource/action | Typically one |
| Response shape | Fixed per endpoint | Client-specified per query |
| Over/under-fetching | Common problem | Solved by design |
| Caching | Native HTTP caching (GET + URL + Cache-Control) | Harder — single POST endpoint; needs client-side or persisted-query caching |
| Versioning | Often via URL (`/v1`, `/v2`) | Convention: add fields, deprecate old ones, avoid versioning |
| Rate limiting | Simple, per-request | Needs query complexity/depth analysis (a query can nest arbitrarily) |
| Tooling/docs | Often hand-maintained, can drift | Auto-generated from schema (introspection, autocomplete) |
| Main risk | Over/under-fetching, endpoint sprawl | N+1 queries, harder caching/rate limiting |

## The N+1 Problem, Concretely

- Query: get 20 users and each user's orders.
- Naive resolver approach: 1 query for the 20 users + 20 separate queries (one per user) for their orders = 21 queries.
- Batched (DataLoader) approach: 1 query for the 20 users + 1 batched query ("get orders for these 20 user IDs") = 2 queries.

## When GraphQL Earns Its Complexity

| Signal | Favors |
|---|---|
| Multiple independently-evolving clients (iOS, Android, web, TV) with different data needs | GraphQL |
| Single client type, simple, stable data needs | REST |
| Heavy reliance on HTTP/CDN caching | REST |
| Frequent screen/feature changes needing different field combinations | GraphQL |
| Public API for external third parties needing simple, cacheable, well-understood semantics | REST |

## Key Numbers / Facts

- GraphQL was developed internally at Facebook starting in 2012 and open-sourced in 2015.
- A GraphQL schema is introspectable — clients and tools (like GraphiQL) can query the schema itself to generate docs, autocomplete, and type-checked queries.

## Summary

- REST's fixed-shape endpoints create over-fetching and under-fetching; GraphQL lets the client specify exactly the data shape it needs from one endpoint and schema.
- The N+1 query problem is GraphQL's core operational risk, addressed with batching/DataLoader patterns.
- GraphQL trades away REST's free HTTP caching and simple rate limiting for flexibility — worth it specifically when multiple, differently-shaped clients evolve against one shared backend.
