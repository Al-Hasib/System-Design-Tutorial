# Microservices Communication & Service Discovery

**Difficulty:** Advanced
**Estimated length:** 16-20 min
**Prerequisites:**
- [30 - Monolith vs Microservices](../30-monolith-vs-microservices/README.md)
- [09 - API Gateway and BFF Pattern](../../Module-02-Networking-and-Communication/09-api-gateway-and-bff-pattern/README.md)
- [20 - Message Queues: Kafka vs RabbitMQ](../../Module-05-Messaging-and-Asynchronous-Systems/20-message-queues-kafka-vs-rabbitmq/README.md)

## Learning Objectives

By the end of this video, you should be able to:

- Distinguish synchronous (REST/gRPC) from asynchronous (message-queue/event-based) inter-service communication and know when to use each.
- Explain the problem service discovery solves and why static configuration breaks down in dynamic environments.
- Compare client-side discovery, server-side discovery, and service-mesh-based discovery.
- Describe how a service registry (like Consul or Eureka) works, including health checks and registration/deregistration.
- Identify where API gateways, load balancers, and service discovery fit together in a real request path.

## Script

### Hook / Intro

Imagine you just split your monolith into twenty microservices, exactly like we discussed last video. Congratulations — now the order service needs to call the inventory service to check stock. Quick question: what IP address does it call? In a world of auto-scaling groups, container orchestrators like Kubernetes, and instances that get created and destroyed constantly, that "IP address" changes every few minutes. Hardcoding it isn't just inconvenient, it's structurally impossible. This is the problem service discovery exists to solve, and it's paired with an equally important question: once you know who to call, how should services actually talk to each other? Let's dig into both.

### Communication Styles: Synchronous vs Asynchronous

There are two broad families of inter-service communication.

**Synchronous, request/response communication** — think REST over HTTP, or gRPC for more performance-sensitive, strongly-typed internal calls. The order service calls the inventory service and blocks waiting for a response: "do you have this item in stock, yes or no?" This is intuitive and mirrors how we think about function calls, and it's a natural fit when the caller genuinely needs an immediate answer to proceed. The cost is coupling: if the inventory service is slow or down, the order service is now slow or down too, unless you protect the call with the timeout, retry, and circuit breaker patterns from Module 6.

**Asynchronous, event-driven communication** — the order service publishes an "OrderPlaced" event to a message queue or broker like Kafka or RabbitMQ, which we covered in Module 5, and moves on without waiting. The inventory service, the notification service, and the analytics service all subscribe to that event and react independently, in their own time. This decouples the services completely in time — if the notification service is down for five minutes, events just queue up and get processed when it recovers, nothing is lost, and the order service was never blocked waiting on it.

The practical rule of thumb: use synchronous calls when you need an answer right now to continue the current operation — checking real-time stock before confirming a purchase. Use asynchronous events for anything that's a reaction to something that already happened and doesn't need to block the original flow — sending a confirmation email, updating a recommendation model, logging an audit trail. Most real systems use a mix of both.

### The Problem: Why You Need Service Discovery

In a static world, you'd put the inventory service's address in a config file and call it a day. But real microservice deployments are dynamic: instances scale up and down based on load, unhealthy instances get killed and replaced, deployments roll new versions in and old versions out continuously, and in Kubernetes a pod's IP address is essentially disposable and changes on every restart. If the order service hardcoded IPs, it would break constantly. Service discovery is the mechanism by which a service can ask, at runtime, "give me a healthy address for the inventory service" — and get a correct, up-to-date answer, every time.

### Patterns of Service Discovery

There are three common patterns, and it's worth knowing all three because you'll see all three in the wild.

**Client-side discovery.** The calling service — the client — queries a service registry directly to get a list of healthy instances of the target service, then picks one itself, often using a client-side load-balancing algorithm like round robin. Netflix's Eureka, combined with their Ribbon client-side load balancer, is the textbook example of this pattern. The advantage is that the client has full visibility and flexibility over load-balancing decisions. The disadvantage is that every client, in every language, needs discovery-aware logic baked in, which is a maintenance burden across a polyglot fleet.

**Server-side discovery.** The client just calls a single, stable endpoint — a load balancer or router — and that intermediary is the one that queries the registry and forwards the request to a healthy instance. AWS ELB combined with ECS service discovery, or Kubernetes' built-in Service abstraction and kube-proxy, are classic examples: your pod just calls `inventory-service.default.svc.cluster.local`, and Kubernetes transparently routes it to a healthy pod behind the scenes. This is simpler for client applications — they don't need any discovery logic at all — but it adds a network hop and makes the load balancer itself a piece of critical infrastructure.

**Service mesh discovery.** A service mesh like Istio or Linkerd deploys a lightweight proxy — a "sidecar," commonly Envoy — alongside every single service instance. All network traffic in and out of a service goes through its sidecar, and the sidecars handle discovery, load balancing, retries, timeouts, mutual TLS encryption, and observability transparently, completely outside your application code. This is the most powerful and most operationally complex option, and it's what very large, mature microservice deployments tend to adopt because it centralizes cross-cutting concerns instead of reimplementing them per service.

### How a Service Registry Actually Works

Regardless of the pattern, there's usually a central component: the service registry — Consul, Eureka, or etcd/Kubernetes' internal API server acting as one. The lifecycle looks like this: when a new service instance starts up, it registers itself with the registry, announcing "I am inventory-service, I'm at this address, here's my health check endpoint." The registry then periodically calls that health check — a simple `/health` HTTP endpoint, typically — and if an instance stops responding or reports unhealthy, the registry marks it as down and stops handing its address out to callers. When an instance shuts down gracefully, it deregisters itself; if it crashes, the failed health checks eventually catch it. This combination of registration, health checking, and deregistration is what keeps the pool of addresses callers receive always pointing at instances that can actually serve traffic.

### Real-World Example

Netflix is the canonical case study for this entire topic. They run thousands of service instances that scale up and down constantly with traffic. They built Eureka specifically so that any service could ask "who are the healthy instances of the recommendation service right now?" and get a live answer, paired with Ribbon for client-side load balancing and Hystrix — the ancestor of today's circuit breaker libraries — for resilience. Today, many teams achieve the same outcome more simply by running on Kubernetes, which bakes server-side service discovery directly into its Service and DNS abstractions, or by layering a service mesh like Istio on top for even more control over traffic management and security between services.

### Recap

To summarize: microservices talk to each other either synchronously, when the caller needs an immediate answer, or asynchronously through events and queues, when they just need to react to something that happened. Because instances come and go constantly, hardcoded addresses don't work — you need service discovery, implemented as client-side discovery (the caller queries the registry and picks an instance), server-side discovery (a load balancer does that on the caller's behalf), or a full service mesh with sidecar proxies handling it transparently. Underneath all three sits a service registry doing continuous health checking so that only healthy instances ever get returned.

### What's Next

We've now covered how to structure services and how they find and talk to each other. But there's a deeper question we've been dancing around this whole module: how do you actually decide where one service ends and another begins? What determines that "inventory" and "orders" should be separate services in the first place, rather than "users" and "billing"? That's a design problem, not just an infrastructure problem, and in the next video we'll introduce Domain-Driven Design — a methodology for drawing those boundaries around real business concepts instead of arbitrary technical splits.

## Key Takeaways

- Use synchronous calls (REST/gRPC) when the caller needs an immediate answer; use asynchronous events/queues when a service just needs to react to something that already happened.
- Service discovery exists because instance addresses in modern deployments (autoscaling, containers, Kubernetes) are dynamic and short-lived.
- Client-side discovery puts registry lookup and load-balancing logic in the caller (e.g., Netflix Eureka + Ribbon).
- Server-side discovery hides that logic behind a load balancer or platform feature (e.g., Kubernetes Services).
- A service mesh (Istio, Linkerd) handles discovery, load balancing, retries, and security via sidecar proxies, transparently to application code.
- A service registry works via continuous registration, health checking, and deregistration to keep only healthy addresses discoverable.
