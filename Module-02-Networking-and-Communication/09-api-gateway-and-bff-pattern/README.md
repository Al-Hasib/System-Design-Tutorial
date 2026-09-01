# API Gateway & Backend-for-Frontend Pattern

**Difficulty:** Intermediate
**Estimated length:** 12-14 min
**Prerequisites:** [08 - Forward Proxy vs Reverse Proxy](../08-forward-proxy-vs-reverse-proxy/README.md), [07 - Load Balancing Explained](../07-load-balancing-explained/README.md)

## Learning Objectives

- Explain what problem an API Gateway solves in a microservices architecture.
- List the core responsibilities of an API Gateway: routing, auth, rate limiting, aggregation, transformation.
- Describe the Backend-for-Frontend (BFF) pattern and why different clients might need different API shapes.
- Compare a plain reverse proxy, an API Gateway, and a BFF.
- Recognize the tradeoffs and failure modes an API Gateway introduces (single point of failure, added latency).

## Script

### Hook / Intro

In the last video we established that a reverse proxy sits in front of your servers and hides them from the client. Now imagine you don't have one backend service — you have thirty. A modern application built on microservices might have a user service, an order service, a payments service, a recommendations service, and dozens more, each independently deployable. If a mobile app had to know the address of all thirty services, individually authenticate with each one, and individually handle each one's quirks, that would be a nightmare to build and an even bigger nightmare to change. This is exactly the problem the API Gateway solves — a single, smart front door for an entire microservices architecture. And once we understand gateways, we'll look at a pattern that often sits right alongside them: the Backend-for-Frontend, or BFF.

### The Problem: Too Many Services, Too Many Clients

Let's set the scene properly. In a microservices architecture, each service usually owns its own data and exposes its own API, and services are deployed and scaled independently — this is great for team autonomy and scalability, which we'll dig into fully in Module 7. But it creates a coordination problem at the edge: every client — a web app, an iOS app, an Android app, a third-party partner integration — now potentially needs to talk to many different services to build a single screen. Worse, every one of those services would need to duplicate cross-cutting concerns: authentication, rate limiting, logging, request validation. That's both wasteful and dangerous — inconsistent auth enforcement across thirty independently-maintained services is a security incident waiting to happen.

### What an API Gateway Does

An API Gateway is a single entry point that sits in front of all of your backend microservices, and it centralizes exactly those cross-cutting concerns so individual services don't have to reimplement them. Let's go through its core jobs one at a time.

**Routing.** The gateway receives every incoming request and routes it to the correct backend service based on the URL path, headers, or other rules — `/users/*` goes to the user service, `/orders/*` goes to the order service. This is the reverse-proxy foundation we covered last video.

**Authentication and authorization.** Instead of every microservice independently validating a JWT or session token, the gateway does it once, centrally, and either rejects unauthorized requests immediately or forwards a verified identity downstream. This dramatically reduces the attack surface and the chance of an inconsistent security implementation somewhere in the fleet.

**Rate limiting and throttling.** The gateway can enforce how many requests a given client or API key is allowed to make, protecting backend services from being overwhelmed — by an abusive client or simply organic traffic spikes — before that traffic even reaches them. We'll cover the actual algorithms behind rate limiting in depth later in this course.

**Request/response transformation.** The gateway can reshape requests and responses — converting protocols (say, translating a REST call at the edge into a gRPC call internally), adding or stripping headers, or reformatting a payload — so that internal services and external clients don't need to agree on identical formats.

**Aggregation.** Perhaps the most powerful capability: a single client request can be fanned out by the gateway into multiple internal calls to different services, with the results combined into one response. Instead of a mobile app making five separate round trips over a potentially slow cellular connection, it makes one request to the gateway, and the gateway does the fan-out internally, over fast, low-latency internal network links.

**Observability.** Centralized logging, metrics, and tracing at the gateway gives you a single place to monitor traffic across your entire service fleet.

Popular real-world implementations include Kong, Amazon API Gateway, Apigee, and Netflix's Zuul — and many teams also build gateway capabilities into a general-purpose reverse proxy like NGINX or an Envoy-based service mesh edge.

### The Tradeoffs

Nothing here is free. An API Gateway becomes a critical, high-traffic component — if it goes down, potentially your entire system becomes unreachable, so it must be built with the same redundancy principles we discussed for load balancers. It adds a network hop and processing overhead, meaning added latency to every single request. And if you're not careful, a gateway can accumulate too much business logic over time, turning into an unintentional monolith at the edge of your microservices — a classic anti-pattern. The discipline is to keep gateway logic to cross-cutting, generic concerns and leave actual business logic inside the services.

### The Backend-for-Frontend (BFF) Pattern

Here's a related but distinct idea. Different client types often have very different needs from the same backend capabilities. A mobile app on a slow, high-latency cellular connection wants a small number of highly aggregated, minimal-payload responses to conserve bandwidth and battery. A desktop web app on a fast connection might want more granular data, more calls, and a richer payload it can render flexibly. A third-party partner API might need a completely different data shape and authentication scheme than either of those.

Trying to satisfy all of these needs with one single, generic gateway API often leads to bloated endpoints stuffed with optional fields and conditional logic to serve every client — a mess to maintain. The Backend-for-Frontend pattern solves this by creating a dedicated, thin backend layer *per client type* — a mobile BFF, a web BFF, a partner BFF — each one tailored exactly to what that specific client needs, aggregating and shaping calls to the underlying services in the way that's ideal for that consumer, while the underlying microservices themselves stay generic and reusable. Each BFF is typically owned by the team that owns that client experience, which also improves team autonomy — the mobile team can iterate on their BFF without waiting on a shared, generic gateway team.

The relationship between the two: an API Gateway is often still the outermost entry point handling universal concerns like TLS and top-level auth, and it may route traffic to the appropriate BFF, which then does client-specific aggregation and shaping before calling the actual backend services.

### Real-World Example

Think of a social media app's "profile page" screen. On mobile, that screen needs a condensed user summary, a small number of recent posts, and follower/following counts — all in one lightweight payload to load fast on a phone. On desktop web, the same screen might show a richer activity feed, detailed analytics, and more embedded media. Under the hood, both need data from a user service, a posts service, and a social-graph service. Rather than the mobile and web clients each making three separate calls and stitching results together themselves, a mobile BFF makes those three calls internally and returns one tailored, minimal JSON blob, while a separate web BFF makes the same three calls but returns a richer, more detailed payload suited to a bigger screen and faster connection. Both BFFs sit behind a shared API Gateway that handles the initial auth check and TLS termination for every request, regardless of which client it came from.

### Recap

An API Gateway is the single front door to a microservices architecture, centralizing routing, authentication, rate limiting, transformation, aggregation, and observability so individual services don't reimplement them. It must be built redundantly since it becomes a critical dependency for the entire system, and it should stay disciplined about not absorbing real business logic. The Backend-for-Frontend pattern complements this by giving each distinct client type its own tailored, thin backend layer, so one-size-fits-all API responses don't become a bottleneck for very different consumers.

### What's Next

We've now covered the full request/response path: HTTP as the language, load balancers and proxies distributing and shielding traffic, and gateways/BFFs orchestrating microservices at the edge. But not everything fits into a simple request-response model — some features need the server to push data to the client the instant something happens. In the next video, we'll cover WebSockets, long polling, and Server-Sent Events — the three main techniques for building real-time features, and how to choose between them.

## Key Takeaways

- An API Gateway is the single entry point to a microservices architecture, centralizing routing, auth, rate limiting, transformation, aggregation, and observability.
- Centralizing cross-cutting concerns at the gateway avoids every microservice reimplementing (and potentially misimplementing) them independently.
- An API Gateway is a critical, high-traffic component that must be made redundant, and it should avoid absorbing real business logic over time.
- The Backend-for-Frontend (BFF) pattern gives each client type (mobile, web, partner) its own tailored backend layer, avoiding one bloated generic API trying to serve everyone.
- A gateway and BFFs often work together: the gateway handles universal concerns at the very edge, then routes to the appropriate client-specific BFF for aggregation and shaping.
