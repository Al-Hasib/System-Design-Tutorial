# Why This Topic Matters: Domain-Driven Design Basics

> **In one sentence:** Microservices only deliver independence if the boundaries are drawn in the right places, and DDD is the only widely used discipline for finding those places — draw them by technical layer instead and you get a distributed monolith.

## The World Before This Idea

A team decides to split a monolith. With no method for deciding boundaries, they use the two most available heuristics.

**Split by technical layer:** a data-access service, a business-logic service, an API service. Now every single feature change touches all three, requiring three deploys in a fixed order across three teams. The coupling is worse than the monolith, and now it runs over the network.

**Split by database table:** a User service, an Order service, a Product service. This looks reasonable until you notice that "Product" means something different to the catalog (name, images, description), to pricing (cost, margin, discounts), to inventory (SKU, warehouse location, stock), and to shipping (weight, dimensions). One shared Product model has to satisfy all four, so it accumulates forty fields, half of which are null in any given context. Every team needs changes to it. It becomes a coordination bottleneck — the exact opposite of independence.

## The Problems It Solves

### 1. The god model that every team must change
**What you see:** A `User` or `Product` class with sixty fields, owned by everyone and therefore by no one, requiring cross-team review for any change.

**Why it happens:** An attempt to build one canonical model of a concept that genuinely has different meanings in different parts of the business.

**How bounded contexts solve it:** Accept that the same word means different things in different contexts, and let each context own its own model. "Customer" in sales is a lead with a pipeline stage; in billing it is a payment method and an address; in support it is a ticket history. Three models, three owners, no shared class. This feels like duplication and is actually decoupling — and it is the single most valuable idea in DDD.

### 2. Boundaries that force distributed transactions
**What you see:** Nearly every business operation spans three services, so you are writing sagas and compensations constantly.

**Why it happens:** Data that changes together was split apart.

**How aggregates solve it:** An aggregate is a cluster of objects that must stay consistent as a unit, with one entry point (the aggregate root) and a transactional boundary around it. Getting aggregates right means most operations are a local transaction inside one service. If you find yourself needing a saga for a routine operation, that is strong evidence the boundary is wrong — and moving the boundary is almost always cheaper than living with the saga.

### 3. Engineers and domain experts talking past each other
**What you see:** A requirements meeting where "order" means the shopping cart to one person, the paid transaction to another, and the shipment to a third. The resulting code encodes the misunderstanding.

**Why it happens:** No shared, precise vocabulary, and translation happening silently in people's heads.

**How ubiquitous language solves it:** One vocabulary, agreed within a bounded context, used identically in conversation, documentation, and code. If the business says "fulfillment," the class is not named `ShippingProcessor`. This sounds like a soft practice; in effect it removes an entire recurring category of defect caused by translation error.

### 4. Optimizing the wrong parts of the system
**What you see:** Enormous engineering effort spent on a beautifully abstracted notification framework, while the pricing engine — the thing the business actually competes on — is a pile of conditionals nobody wants to touch.

**Why it happens:** No explicit distinction between what differentiates the business and what merely supports it.

**How subdomain analysis solves it:** Identify the **core domain** — the part that is the reason the company wins — and put your best people and deepest modeling there. Supporting subdomains get straightforward implementations. Generic subdomains (auth, billing, email) get bought, not built. This single piece of triage often redirects more engineering value than any technical decision.

## The Price You Pay

- **It requires access to domain experts.** DDD is fundamentally a conversation between engineers and people who understand the business. Without that access, you get the ceremony without the insight.
- **It is heavy for simple domains.** A CRUD application does not need aggregates, repositories, and value objects. Applying the full tactical toolkit to a shallow domain produces ten files where one would do, and it is the most common way DDD earns a bad reputation.
- **Deliberate duplication is uncomfortable.** Having three `Customer` models feels wrong to engineers trained on DRY. It takes discipline to accept that duplicating a *model* to avoid coupling *teams* is usually the right trade — and judgment to know when it is not.
- **Boundaries are wrong on the first attempt.** You rarely understand a domain well enough up front. This is another strong argument for the modular monolith: get the boundaries wrong cheaply inside one codebase, learn, adjust, and only then extract services.
- **The vocabulary is large.** Entities, value objects, aggregates, repositories, factories, domain events, anti-corruption layers, context maps. Teams can spend more energy arguing about terminology than modeling the domain.

## When You Need It — and When You Don't

| Apply DDD when | Keep it simple when |
|---|---|
| The domain is genuinely complex (finance, logistics, healthcare, insurance) | It is CRUD over forms |
| You are deciding microservice boundaries | It is a prototype or a short-lived tool |
| Multiple teams must own distinct areas | One small team owns everything |
| Business rules change often and must be locatable | Rules are trivial and stable |

Even when the tactical patterns are overkill, **bounded contexts and ubiquitous language are almost always worth it** — they are cheap, and they prevent the most expensive mistakes.

## Why This Shows Up in Interviews

You are unlikely to be asked to explain DDD terminology. You are very likely to be asked to decompose a system into services — and *how you justify the boundaries* is what is being graded. Saying "I would separate these by business capability: order management, inventory, payments, and fulfillment, because each has a distinct model and a distinct owner, and I have drawn them so that a normal checkout does not require a distributed transaction" is a strong answer. Splitting by technical layer, or by database table, is a notable weakness. Mentioning bounded contexts by name, and that the same term can legitimately mean different things in different services, is a clear depth signal.

## How It Connects

DDD is the method that makes **monolith vs microservices** (topic 30) a solvable problem rather than a guess, and good boundaries directly reduce the need for **distributed transactions** (topic 28) and chatty **inter-service communication** (topic 31). Domain events are the natural payload of **event-driven architecture** (topic 22) and **Pub/Sub** (topic 21). Aggregate boundaries often make good **shard keys** (topic 14), and bounded contexts frequently map to **BFF** boundaries (topic 9).

**Next:** [Transport Protocols: TCP vs UDP & gRPC](../../Module-08-Protocols-Formats-and-Security/33-transport-protocols-tcp-udp-and-grpc/why.md) — dropping down to the layer all this communication actually runs on.
