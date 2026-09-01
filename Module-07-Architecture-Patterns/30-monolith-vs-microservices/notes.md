# Notes: Monolith vs Microservices

## Definitions

- **Monolith**: A system built and deployed as a single unit/artifact. Internally may be layered or modular, but all modules share one process (or identical replicas of it) and typically one database.
- **Modular monolith**: A monolith whose internal modules have clear boundaries and interfaces (and often separate schemas within one database) but still ship as one deployable unit. A common stepping stone toward microservices.
- **Microservices**: A system decomposed into independently deployable services, each owning its own data store, communicating over the network (synchronous RPC/REST/gRPC or asynchronous messaging).
- **Strangler fig pattern**: An incremental migration strategy — put a facade/gateway in front of a monolith and gradually replace pieces of functionality with new services, redirecting traffic bit by bit, instead of a risky rewrite.
- **Conway's Law**: Systems tend to mirror the communication structure of the organizations that build them. Microservices are often as much an org-design tool as a technical one.

## Comparison Table

| Dimension | Monolith | Microservices |
|---|---|---|
| Deployment unit | Single artifact/process | Many independent services |
| Data ownership | Usually one shared database | Each service owns its own data store |
| Transactions | Easy ACID transactions | Distributed transactions (saga, 2PC) needed |
| Scaling | Scale the whole app together | Scale each service independently |
| Fault isolation | One crash can take down everything | A failing service can be isolated (with circuit breakers) |
| Team autonomy | Coordination needed across one codebase | Teams can own/deploy services independently |
| Technology choice | Usually one stack for everything | Polyglot — different languages/DBs per service |
| Operational complexity | Low (one pipeline, one set of logs) | High (service discovery, distributed tracing, orchestration) |
| Testing | Simple end-to-end tests | Harder — need contract tests, integration environments |
| Debugging | Single stack trace, single log stream | Requires distributed tracing/correlation IDs |
| Good fit for | Small-to-mid teams, early-stage products, strong consistency needs | Large orgs, independently-scaling components, many autonomous teams |

## When to Prefer Each

**Favor a monolith / modular monolith when:**
- The team is small (roughly one to a few teams).
- The product/domain boundaries are still unclear (early-stage startup).
- You need strong transactional consistency across most operations.
- You want to minimize operational/infra overhead.

**Favor microservices when:**
- Multiple teams need to deploy independently without blocking each other.
- Different components have very different scaling or resource profiles.
- You need failure isolation between components for availability.
- You have (or are willing to build) the platform maturity for CI/CD, observability, and service discovery per service.

## Bullet Summary

- The defining trait of a monolith is the deployment boundary, not the internal code quality.
- Microservices convert in-process function calls into network calls — with all the failure modes (latency, partial failure, timeouts) that implies.
- Distributed transactions across microservices require patterns like Saga or 2PC (see Module 6) instead of native ACID.
- Start with a modular monolith by default; migrate via the strangler fig pattern when concrete scaling/organizational signals appear.
- Segment's 2018 "goodbye microservices" post-mortem is the canonical cautionary tale about adopting microservices without a matching problem.
