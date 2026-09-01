# Study Notes: API Gateway & Backend-for-Frontend

## Definitions

- **API Gateway:** A single entry point in front of a microservices architecture that centralizes routing, authentication, rate limiting, transformation, aggregation, and observability.
- **Backend-for-Frontend (BFF):** A dedicated, thin backend layer built per client type (mobile, web, partner) that aggregates and shapes calls to underlying services specifically for that client's needs.
- **Aggregation:** Combining multiple internal service calls into a single response for the client, reducing client-side round trips.
- **Cross-cutting concern:** Functionality needed across many services (auth, logging, rate limiting) that's better centralized than duplicated per service.

## Reverse Proxy vs API Gateway vs BFF

| Aspect | Reverse Proxy | API Gateway | BFF |
|--------|----------------|--------------|-----|
| Primary purpose | Hide/protect backend servers, basic routing/TLS | Unified entry point for microservices with cross-cutting logic | Client-specific API shaping and aggregation |
| Awareness of business/API semantics | Low (mostly transport/HTTP level) | Medium-high (routes, auth, rate limits per API) | High (tailored to one client's exact needs) |
| Per-client customization | None | Usually uniform across all clients | Fully customized per client type |
| Typical owner | Infra/platform team | Platform/API team | Team that owns that client (e.g., mobile team) |
| Examples | NGINX, HAProxy | Kong, Amazon API Gateway, Apigee, Netflix Zuul | Custom Node/Go/Java service per client type |

## Core API Gateway Responsibilities

- Routing requests to the correct backend service (path/header-based)
- Authentication & authorization (centralized token/session validation)
- Rate limiting & throttling per client/API key
- Request/response transformation (protocol translation, payload reshaping)
- Aggregation (fan-out to multiple services, combine into one response)
- Centralized observability: logging, metrics, tracing

## Why BFF Exists

- Different clients have different constraints: mobile (bandwidth/battery-limited, wants minimal aggregated payloads), web (richer payloads, more granular calls), partner APIs (different auth/data shape entirely).
- A single generic API trying to serve all clients tends to become bloated with optional fields and conditional logic.
- BFF isolates client-specific logic into its own layer owned by the team responsible for that client, improving autonomy and reducing coordination overhead.

## Tradeoffs / Failure Modes

- **Single point of failure:** Gateway sits in the path of all traffic — must be deployed redundantly (same principle as load balancers).
- **Added latency:** Every request passes through an extra hop and processing step.
- **Gateway sprawl / "edge monolith" anti-pattern:** Business logic creeping into the gateway over time makes it a bottleneck for change and a shared point of coupling across teams.
- **BFF duplication:** Multiple BFFs may duplicate similar aggregation logic — acceptable tradeoff for the autonomy gained, but worth monitoring.

## Summary

- API Gateway = single smart front door for a microservices architecture; centralizes cross-cutting concerns.
- BFF = a dedicated, tailored backend per client type, often sitting behind or alongside the gateway.
- Both build on the reverse-proxy foundation but add application/business awareness the plain reverse proxy doesn't have.
