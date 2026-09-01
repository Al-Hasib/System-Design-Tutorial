# Notes: Domain-Driven Design Basics

## Definitions

- **Domain-Driven Design (DDD)**: A design approach (Eric Evans, 2003) where software structure mirrors the business domain, built through close collaboration between engineers and domain experts.
- **Domain**: The subject area / business the software serves (e.g., "e-commerce fulfillment", "insurance underwriting").
- **Ubiquitous language**: A precise, shared vocabulary between engineers and domain experts, used consistently in conversation, documentation, and code within a given context.
- **Bounded context**: An explicit boundary within which a specific model and its ubiquitous language apply consistently and unambiguously. The same real-world term (e.g., "Customer") can mean different things in different bounded contexts.
- **Anti-corruption layer (ACL)**: A translation layer at the boundary between two bounded contexts that maps one context's model/vocabulary to another's, preventing one model from "leaking into" and corrupting the other.
- **Aggregate**: A cluster of domain objects treated as a single consistency unit within a bounded context (e.g., an `Order` and its `OrderLineItems`).
- **Core domain**: The part of the business that provides competitive differentiation — deserves the most design investment.
- **Supporting subdomain**: Necessary for the business but not a differentiator — build simply.
- **Generic subdomain**: Common to any business, not differentiating — prefer buying/using existing solutions.

## Bounded Context vs Microservice

| Concept | Description |
|---|---|
| Bounded context | A modeling boundary — where a specific ubiquitous language and model apply |
| Microservice | A deployment/runtime boundary — an independently deployable unit |
| Relationship | A bounded context is a strong candidate to become one microservice (or a small group of tightly related ones); it gives a principled reason for where the service boundary should be, instead of an arbitrary technical split |

## Subdomain Classification & Strategy

| Subdomain type | Differentiates the business? | Example | Engineering strategy |
|---|---|---|---|
| Core domain | Yes | Airbnb's search & matching, a retailer's recommendation engine | Invest heavily; best engineers; deep custom modeling |
| Supporting subdomain | No, but necessary | Custom warehouse fulfillment logic | Build adequately; keep it simple, no over-investment |
| Generic subdomain | No | Authentication, payment processing, email sending | Buy or use existing solutions (Stripe, Auth0) rather than build |

## Bullet Summary

- DDD's premise: software structure should mirror the real business domain, shaped through ongoing collaboration with domain experts.
- Ubiquitous language prevents one overloaded model (e.g., a single `Customer` class) from trying to serve every team's different meaning of a term.
- A bounded context is the boundary where one model/language is valid — it's the natural unit to map to a microservice.
- Don't share a database/model across bounded contexts; translate between them explicitly via an anti-corruption layer.
- Classify subdomains as core (invest deeply), supporting (keep lean), or generic (buy, don't build) to prioritize engineering effort.
