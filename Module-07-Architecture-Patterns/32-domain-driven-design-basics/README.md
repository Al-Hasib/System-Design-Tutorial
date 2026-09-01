# Domain-Driven Design Basics for System Design

**Difficulty:** Advanced
**Estimated length:** 14-18 min
**Prerequisites:**
- [30 - Monolith vs Microservices](../30-monolith-vs-microservices/README.md)
- [31 - Microservices Communication & Service Discovery](../31-microservices-communication-and-service-discovery/README.md)

## Learning Objectives

By the end of this video, you should be able to:

- Explain what Domain-Driven Design (DDD) is and the problem it was created to solve.
- Define "ubiquitous language" and explain why shared vocabulary matters for system boundaries.
- Define "bounded context" and use it to reason about where one service's responsibility ends and another's begins.
- Distinguish core domain, supporting subdomain, and generic subdomain, and explain why that distinction matters for where to invest engineering effort.
- Apply DDD concepts to decide microservice boundaries in a concrete example.

## Script

### Hook / Intro

In the last two videos, we talked about splitting a monolith into microservices, and how those services find and talk to each other. But I deliberately skipped over the hardest question of all: how do you actually decide where the boundary between "order service" and "inventory service" should go? Why isn't "inventory" split further into "warehouse-inventory" and "store-inventory"? Get this wrong, and you end up with services that constantly need to call each other for basically every operation — which is really just a distributed monolith with extra latency and more ways to fail. Domain-Driven Design, or DDD, is the methodology that gives you a principled answer to this question, and today we'll cover its core ideas.

### What Is Domain-Driven Design?

Domain-Driven Design comes from Eric Evans' 2003 book of the same name. Its central premise is refreshingly simple to state, if not always simple to do: the structure of your software should mirror the structure of the business domain it serves, and that structure should be built through close, continuous collaboration between engineers and domain experts — the people who actually understand the business, like warehouse managers, underwriters, or logistics coordinators — rather than being designed in isolation by engineers guessing at how the business works.

DDD isn't primarily a technical pattern — it's a design discipline. Its two most important exports into system design specifically are the concept of "ubiquitous language" and the concept of "bounded context," and those two ideas are exactly what let you draw sane microservice boundaries.

### Ubiquitous Language

Here's a problem every engineer has hit: the word "customer" means something different to the sales team than it does to the support team, which means something different again to the billing team. Sales might mean "anyone who's shown interest," support might mean "anyone with an active account," and billing might mean "anyone with a valid payment method on file." If your codebase has one giant `Customer` class trying to satisfy all three meanings, it turns into an unmaintainable mess of optional fields and conditional logic, because you're forcing three different concepts to share one name.

Ubiquitous language is the discipline of building a precise, shared vocabulary with domain experts for a specific part of the business, and then using those exact terms — consistently — in conversations, documentation, and, critically, in the code itself: class names, method names, API fields. If the domain experts say "a Lead becomes a Customer once they've made a first purchase, and becomes an Account when they enter a support contract," your code should have `Lead`, `Customer`, and `Account` as distinct concepts, not one overloaded `Customer` object with five boolean flags. This sounds like a small thing, but it's the foundation everything else is built on — it forces you to actually understand the domain instead of assuming you do.

### Bounded Context

Here's the key insight that follows directly from ubiquitous language: since "customer" genuinely means different things in different parts of the business, you shouldn't try to build one unified model of "customer" that works everywhere. Instead, you define a bounded context — an explicit boundary within which a particular model and its ubiquitous language apply consistently and unambiguously. Inside the "Sales" bounded context, `Customer` means one specific thing, with one specific set of fields and behaviors. Inside the "Billing" bounded context, `Customer` can mean something else entirely, modeled completely differently, even though it's the same real-world person.

This is, almost one-to-one, how you should be thinking about microservice boundaries. A bounded context is a fantastic candidate to become a microservice, precisely because it's a self-contained unit of meaning that doesn't need to reach into another context's internal model to do its job. When two bounded contexts do need to reference each other's concepts — Billing needs to know about the Sales-context Customer to send an invoice — you don't share the model directly; you define an explicit translation, sometimes called an "anti-corruption layer," that maps between the two contexts' vocabularies at their boundary. This is exactly why microservices should own their own data and communicate through well-defined APIs or events rather than sharing a database — that shared database is what forces every context into one incoherent shared model in the first place.

### Core, Supporting, and Generic Subdomains

DDD also gives you a way to prioritize your engineering investment. Not every part of your business deserves the same level of design care. Evans distinguishes three kinds of subdomains. The **core domain** is what actually differentiates your business competitively — for an online retailer, that might be the recommendation and personalization engine, or dynamic pricing. This is where you should put your best engineers and the most careful, custom domain modeling, because it's your competitive advantage. A **supporting subdomain** is necessary for the business but not itself a differentiator — order fulfillment logic tailored to your specific warehouse setup, for example. It matters, but you can build it more simply and it doesn't need world-class design. A **generic subdomain** is something every business needs and that doesn't differentiate you at all — authentication, payment processing, sending emails. For generic subdomains, the right move is usually to buy or use an existing well-built solution — Stripe for payments, Auth0 for authentication — rather than build it in-house. This classification directly informs both your architecture and your team-staffing decisions: invest deeply in the core, keep supporting subdomains lean, and outsource generic subdomains entirely when you can.

### Real-World Example

Think about a company like Airbnb. Their core domain is almost certainly the search-and-matching experience — connecting the right guest to the right listing at the right time, with the right pricing and ranking — because that's the actual product differentiator, and you'd expect their best engineering talent concentrated there, with deeply custom modeling of concepts like "listing availability" or "booking request" as bounded contexts of their own. Payments processing, on the other hand, is a generic subdomain — critical to get right, but not something they'd build from scratch when mature providers already solve it well; that's exactly why payment processing is so commonly outsourced to something like Stripe or Braintree in companies across every industry. This mapping — core domain gets custom, deeply-invested bounded contexts, generic subdomains get bought — is DDD's practical output showing up directly in real architecture decisions.

### Recap

Domain-Driven Design says your software structure should mirror your actual business domain, built through close collaboration with domain experts. Ubiquitous language means using one precise, consistent vocabulary — and using those exact terms in your code — within a given part of the business. A bounded context is the explicit boundary within which that vocabulary and model apply, and it's the natural unit to map onto a microservice, with anti-corruption layers translating between contexts at their edges instead of sharing models. And classifying work into core, supporting, and generic subdomains tells you where to invest deep custom engineering effort versus where to keep it simple or just buy an existing solution.

### What's Next

That wraps up Module 7 on architecture patterns — you now have a framework for deciding monolith versus microservices, how those services communicate and discover each other, and how to actually decide where their boundaries should go using DDD. Before we get to full case-study interviews, Module 8 drops down a level to fill in a few foundational pieces we've been assuming so far: how transport protocols like TCP, UDP, and gRPC actually move bytes, what's happening inside a web server under load, how services agree on a message format, and the security fundamentals every backend needs. Then, in Module 9, we shift from patterns to practice: we'll start walking through full end-to-end system design interview questions, starting with designing a URL shortener, and you'll see every concept from this course get applied together in one place.

## Key Takeaways

- DDD's central idea: software structure should mirror the real business domain, developed in close collaboration with domain experts.
- Ubiquitous language means using one precise, consistent vocabulary — reflected directly in code — within a specific part of the business, rather than one overloaded shared model.
- A bounded context is the explicit boundary within which a model and its language apply consistently; it's the natural candidate for a microservice boundary.
- Use anti-corruption layers to translate between bounded contexts instead of sharing a model or database across them.
- Classify work as core domain (invest deeply, differentiator), supporting subdomain (necessary, keep lean), or generic subdomain (buy/use existing solutions like Stripe or Auth0) to prioritize engineering effort.
