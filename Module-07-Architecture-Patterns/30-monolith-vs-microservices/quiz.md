# Practice & Interview Questions

**1. What is the actual defining characteristic of a monolith — is it "bad code" or something else?**
The deployment boundary. A monolith is defined by being built and shipped as a single deployable unit (typically with one shared database), not by the quality or organization of its internal code. A monolith can be cleanly modularized internally.

**2. Name three concrete costs of moving to microservices that teams often underestimate.**
- Distributed transactions replace free ACID transactions (need sagas or 2PC).
- Network calls introduce latency and partial-failure modes that in-process calls don't have.
- You need new infrastructure: service discovery, distributed tracing/correlation IDs, and per-service CI/CD and monitoring.

**3. What is a "modular monolith" and why is it a good default starting point?**
A monolith whose internal modules have clear boundaries and interfaces (and sometimes separate schemas) but still deploy as a single unit. It gives most of the maintainability benefits of service separation without the operational overhead of a distributed system, and it makes a future split into microservices easier because the seams already exist.

**4. Describe the strangler fig migration pattern.**
Instead of rewriting a monolith in one risky big-bang effort, you place a facade or API gateway in front of it and incrementally extract individual capabilities into new services, redirecting the relevant traffic to each new service as it's ready, until the monolith is "strangled" down to nothing (or a small core).

**5. When would you NOT recommend microservices to a team?**
When the team is small (roughly one to a few small teams), the domain boundaries aren't well understood yet, strong cross-entity transactional consistency is a frequent requirement, or the org doesn't have the platform maturity (CI/CD, observability, on-call practices) to handle distributed systems operations. In these cases the operational tax of microservices outweighs the benefits.

**6. What is Conway's Law and why is it relevant to this decision?**
Conway's Law states that systems tend to mirror the communication structure of the organizations that build them. It's relevant because microservices adoption is often more about enabling team autonomy (independent deployment, ownership) than about pure technical scaling — you're designing service boundaries to match desired team boundaries.

**7. In an interview, you're asked to design a photo-sharing app for a 6-person startup. Monolith or microservices — and why?**
Monolith (ideally modular). At this scale, one small team benefits far more from simplicity, fast iteration, and low operational overhead than from independent scaling or team autonomy, neither of which is a real constraint yet.

**8. What made Segment revert away from some of their microservices?**
Their microservices had highly correlated load and shared failure modes, so the fault-isolation and independent-scaling benefits never really materialized in practice, yet they paid the full daily operational cost — on-call burden, deployment/coordination complexity — of running hundreds of services.

**9. How does fault isolation differ between a monolith and a microservices architecture?**
In a monolith, an unhandled exception or resource exhaustion (e.g., memory leak) in one module can crash or degrade the entire process, taking down all functionality. In microservices, a failing service can be isolated — especially with circuit breakers and bulkheads (Module 6) — so other services keep functioning, though this requires deliberate design; it isn't automatic.

**10. What signals should you look for as evidence that it's time to split a monolith into services?**
Team growth causing constant merge conflicts and deployment coordination pain; specific components (e.g., video processing) needing very different scaling profiles than the rest of the app; a need for independent failure domains for availability; or distinct teams needing to own and release their slice of the system independently.

**11. Does choosing microservices mean you lose consistency guarantees entirely?**
Not entirely, but you lose native cross-service ACID transactions. You have to explicitly design for consistency using patterns like the Saga pattern (compensating transactions) or, in narrower cases, two-phase commit — both covered in Module 6 — and often accept eventual consistency between services.

**12. Why is "polyglot technology" cited as a microservices advantage, and what's the trade-off?**
Each service can use the language, framework, or database best suited to its specific problem (e.g., Go for a high-throughput service, Python for ML). The trade-off is increased operational diversity — more tooling, more skills needed across the org, and less code/infra reuse between teams.
