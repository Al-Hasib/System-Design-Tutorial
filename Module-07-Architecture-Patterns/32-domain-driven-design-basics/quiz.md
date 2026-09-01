# Practice & Interview Questions

**1. What is the central premise of Domain-Driven Design?**
That software structure should mirror the structure of the real business domain it serves, and that this modeling should be built through close, continuous collaboration between engineers and domain experts, rather than engineers guessing at the domain in isolation.

**2. What is "ubiquitous language" and why does it matter?**
It's a precise, shared vocabulary developed with domain experts for a specific part of the business, used consistently in conversation, documentation, and in the code itself (class names, fields, methods). It matters because without it, teams silently mean different things by the same word (e.g., "Customer"), leading to overloaded, unmaintainable models.

**3. What is a bounded context, and how does it relate to ubiquitous language?**
A bounded context is the explicit boundary within which a particular model and its ubiquitous language apply consistently and unambiguously. Since a term like "Customer" can validly mean different things in different parts of the business, each bounded context is allowed its own consistent definition instead of forcing one universal model.

**4. Why is a bounded context a good candidate to become a microservice?**
Because it's already a self-contained unit of meaning and behavior that doesn't need to reach into another context's internal model to do its job — mapping it to a microservice gives you a boundary based on actual business meaning rather than an arbitrary technical split, reducing the need for services to constantly call each other.

**5. What is an anti-corruption layer and when do you need one?**
It's a translation layer at the boundary between two bounded contexts that maps one context's model/vocabulary to another's. You need one whenever two bounded contexts must reference each other's concepts (e.g., Billing needs Sales' Customer) so that neither context's internal model is forced to conform to, or leak into, the other's.

**6. Distinguish core domain, supporting subdomain, and generic subdomain, with an example of each.**
Core domain differentiates the business competitively and deserves deep investment (e.g., a retailer's recommendation engine). Supporting subdomain is necessary but not differentiating, and can be built simply (e.g., custom warehouse fulfillment logic). Generic subdomain is common to any business and not differentiating at all, so it's usually better to buy/use an existing solution (e.g., authentication via Auth0, payments via Stripe).

**7. In a system design interview, how could you use DDD concepts to justify your microservice boundaries instead of just guessing?**
Identify the bounded contexts by talking through the actual business workflows and where vocabulary/meaning shifts (e.g., "Order" behaves differently in the Cart context vs. the Fulfillment context), and map each bounded context to a service. This gives a principled, defensible rationale ("these are separate models with separate lifecycles") rather than an arbitrary technical split.

**8. Why shouldn't two microservices share the same database, from a DDD perspective?**
A shared database forces both services to use one common schema/model, which contradicts the idea that each bounded context should have its own consistent model and language. Sharing a database is effectively what collapses multiple bounded contexts back into one, defeating the purpose of the split and tightly coupling the services' schemas.

**9. Give an example of when NOT to build something in-house according to DDD's subdomain classification.**
Building your own payment processing or authentication system when you're not a payments/identity company — these are generic subdomains that don't differentiate your business, so it's more efficient to use a mature existing provider (e.g., Stripe, Auth0) than to invest engineering effort building and maintaining them yourself.

**10. How does DDD help decide where to put your best engineers?**
By classifying subdomains: the core domain is your competitive differentiator, so it deserves the most design care and your strongest engineers; supporting subdomains need competent but not exceptional effort; generic subdomains often don't need in-house engineering effort at all since you can buy a solution.

**11. What's the risk of treating one "Customer" model as valid across your entire company's codebase?**
It becomes an unmaintainable object bloated with fields and conditional logic trying to satisfy every team's different definition of "customer" (sales, support, billing, etc.), making changes risky and coupling teams that don't actually need to be coupled.

**12. How does DDD connect to the "modular monolith" idea from video 30?**
Bounded contexts can be used to define the internal module boundaries of a modular monolith even before you split into microservices — each module maps to a bounded context with its own model/vocabulary and (ideally) its own schema, giving you clean seams that make a future strangler-fig migration to microservices far easier.
