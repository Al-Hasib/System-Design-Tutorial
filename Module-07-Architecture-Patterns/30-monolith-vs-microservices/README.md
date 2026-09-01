# Monolith vs Microservices

**Difficulty:** Intermediate/Advanced
**Estimated length:** 15-18 min
**Prerequisites:**
- [04 - Scalability Basics: Vertical vs Horizontal Scaling](../../Module-01-Foundations/04-scalability-basics-vertical-vs-horizontal-scaling/README.md)
- [09 - API Gateway and BFF Pattern](../../Module-02-Networking-and-Communication/09-api-gateway-and-bff-pattern/README.md)

## Learning Objectives

By the end of this video, you should be able to:

- Define what a monolithic architecture and a microservices architecture are, precisely.
- Explain the real trade-offs between the two styles — not just "microservices are modern, monoliths are legacy."
- Identify the organizational and technical signals that suggest one style over the other.
- Describe the "modular monolith" and "strangler fig" pattern as a middle ground and migration path.
- Avoid the most common mistake teams make: adopting microservices before they have the problem microservices solve.

## Script

### Hook / Intro

Quick question: if I told you that Amazon, Shopify, and Stack Overflow all ran — or still run — huge parts of their business on monoliths, would that surprise you? For the last decade, "microservices" has been treated almost like a badge of engineering maturity. But the truth is messier and much more interesting. Today we're going to cut through the hype and talk about monolith versus microservices as an actual engineering trade-off, with real costs on both sides, so that when you're in a design interview — or making this call for real at work — you can reason about it instead of just repeating a buzzword.

### What Is a Monolith?

A monolithic architecture is a system where all the functionality — the user service, the order service, the payment logic, the notification logic — lives in a single codebase and is deployed as a single unit. That doesn't mean it's badly designed. A monolith can absolutely have clean internal modules, well-defined interfaces between them, and a layered architecture. The defining trait isn't "messy code," it's the deployment boundary: one build, one artifact, one process (or a small fleet of identical copies of that one process behind a load balancer), one database, typically.

Think of a monolith like a single restaurant kitchen. Every station — grill, salad, dessert — is under one roof, one head chef coordinates everything, and the whole kitchen opens or closes together. If the dessert station catches fire, the whole kitchen might have to shut down.

### What Are Microservices?

Microservices architecture splits that same functionality into a set of independently deployable services, each owning its own data and its own release cycle, communicating over the network — usually via REST, gRPC, or asynchronous messaging, which we cover in Module 5. Each service is small enough that a single team can fully own it: build it, deploy it, scale it, and be on call for it.

Now it's like a food court instead of one kitchen. The noodle stall, the burger stall, and the smoothie stall are independently run businesses. If the noodle stall has a health inspection issue and closes, the burger stall keeps serving customers. But now you need shared infrastructure — seating, a payment system that works across vendors, coordination between them — none of which existed when it was all one kitchen. That coordination is the cost you're signing up for.

### The Real Trade-offs

Let's be concrete, because this is where interviews are won or lost.

**Monolith advantages:** Simplicity of development — one codebase to clone and run. Simplicity of deployment — one CI/CD pipeline, one thing to roll back. Strong consistency is easy because you likely have one database and can use real ACID transactions. Cross-cutting changes — say, renaming a field used in three modules — are a single pull request, not a coordinated multi-team rollout. Debugging is easier: one process, one set of logs, a single stack trace from HTTP request to database call.

**Monolith drawbacks:** As the codebase and team grow, build times grow, test suites grow, and merge conflicts multiply. You scale the entire application even if only one piece — say, image processing — is the bottleneck, which wastes resources. A bug in one module can crash the entire process. And technology choice is locked in globally — you can't easily use Go for one performance-critical piece and Python for a data science piece.

**Microservices advantages:** Independent deployability — the checkout team can ship ten times a day without coordinating with the search team. Independent, targeted scaling — you scale just the image-processing service under load. Fault isolation — if the recommendation service falls over, checkout can still work, especially with the circuit breaker patterns from Module 6. Teams can pick the right tool for each job, and large organizations can scale by having many small, autonomous teams — this is really an organizational pattern as much as a technical one, often called "Conway's Law" in reverse: you design your system boundaries to match the team boundaries you want.

**Microservices drawbacks, and this is the part people underweight:** You've traded in-process function calls for network calls, which are slower and can fail in new ways — timeouts, partial failures, retries. Distributed transactions become genuinely hard — no more free ACID across services, which is exactly why Module 6 spent time on sagas and two-phase commit. You need service discovery, which is our very next video. You need distributed tracing and centralized logging just to answer "why did this request take 800ms." Your infrastructure and operational complexity — Kubernetes, service meshes, API gateways — grows enormously. And testing an end-to-end flow that touches six services is dramatically harder than testing one monolith.

### How Do You Actually Decide?

Here's the heuristic I'd give you: start with a monolith, and prefer a well-modularized monolith — sometimes called a "modular monolith" — where the code is already split into modules with clear boundaries and no direct database coupling between them, even though it deploys as one unit. This gives you most of the maintainability benefits of microservices without the operational cost, and it makes a future split much easier because the seams already exist.

You move toward microservices when you have concrete signals: your team has grown past what one deployable unit can support without constant merge pain; specific components have wildly different scaling needs — a video transcoding service versus a user-profile service; you need different parts of the system to fail independently for availability reasons; or different teams genuinely need to own and deploy their slice independently. The "strangler fig" pattern is the standard migration path — you put a facade or an API gateway in front of the monolith and gradually peel off and replace individual capabilities as new, independent services, redirecting traffic incrementally, rather than doing a risky big-bang rewrite.

### Real-World Example

Segment, the customer data platform, is a famous case study here. They started as a monolith, migrated aggressively to microservices — hundreds of them — and then in 2018 they publicly wrote about consolidating a chunk of those services back into a monolith. Why? Their microservices had highly correlated load and shared failure modes, so the isolation benefit never materialized, but they paid the full operational tax — on-call burden, deployment complexity — every single day. It's a great reminder that microservices are a tool for a specific set of problems, not a universal upgrade. Contrast that with Amazon or Netflix, where massive scale, hundreds of independent teams, and wildly different service-level requirements make microservices a clear win. The right answer depends entirely on your context.

### Recap

Let's tie it together. A monolith is one deployable unit with typically one shared database; microservices are many independently deployable units, each owning its own data, talking over the network. Monoliths win on simplicity, consistency, and low operational overhead; microservices win on independent scaling, fault isolation, and team autonomy — at the cost of distributed systems complexity. Default to a modular monolith, and split into microservices when you have a concrete organizational or scaling reason to do so, not because it's trendy.

### What's Next

Once you do have microservices, they need to talk to each other and find each other at runtime — a service can't just call `localhost`. In the next video, we'll dig into microservices communication patterns and service discovery: how does the order service actually know the network address of the inventory service, especially when instances are constantly starting up, shutting down, and moving around in a cluster?

## Key Takeaways

- A monolith is defined by its single deployment unit, not by code quality — a monolith can be well-modularized internally.
- Microservices trade simplicity and easy consistency for independent scaling, fault isolation, and team autonomy.
- The biggest hidden cost of microservices is operational: service discovery, distributed tracing, network failure handling, and giving up free ACID transactions across services.
- A "modular monolith" is a strong default that preserves a future migration path via the strangler fig pattern.
- Segment's public monolith-consolidation story shows microservices are a targeted tool, not a default best practice.
