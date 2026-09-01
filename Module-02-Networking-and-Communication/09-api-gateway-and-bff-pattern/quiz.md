# Practice & Interview Questions

**1. What core problem does an API Gateway solve in a microservices architecture?**
It gives clients a single, unified entry point instead of requiring them to know about and individually call dozens of independent services, and it centralizes cross-cutting concerns (auth, rate limiting, logging) so each service doesn't have to reimplement them.

**2. List at least four core responsibilities typically handled by an API Gateway.**
Routing to the correct backend service, authentication/authorization, rate limiting/throttling, request/response transformation, aggregation of multiple service calls, and centralized observability (logging/metrics/tracing) — any four of these.

**3. Why is centralizing authentication at the gateway generally safer than letting each microservice implement its own auth checks?**
It reduces the attack surface and the chance that one of many independently-maintained services implements auth incorrectly or inconsistently; every request is validated the same way, once, before reaching any backend service.

**4. What is "aggregation" in the context of an API Gateway, and why does it matter for mobile clients especially?**
Aggregation is when the gateway fans a single client request out into multiple internal service calls and combines the results into one response. It matters especially for mobile clients on slow/high-latency connections, since it replaces several slow round trips from the client with one, doing the fan-out over fast internal network links instead.

**5. What is the "edge monolith" anti-pattern, and how do you avoid it?**
It's when an API Gateway accumulates real business logic over time instead of staying limited to cross-cutting, generic concerns, effectively becoming an unintentional monolith that couples all services together at the edge. It's avoided by disciplined scope: keep the gateway to routing, auth, rate limiting, and transformation, and leave business logic inside the owning services.

**6. What problem does the Backend-for-Frontend (BFF) pattern solve that a single generic API struggles with?**
Different client types (mobile, web, partner) have very different needs — payload size, number of round trips, data shape. A single generic API trying to satisfy all of them tends to become bloated with optional fields and conditional logic. BFF solves this by giving each client type its own tailored backend layer.

**7. How does a BFF typically relate to the underlying microservices — does it replace them?**
No — the BFF sits between the client and the underlying microservices, calling them and shaping/aggregating their responses specifically for one client type. The underlying services themselves stay generic and reusable across all BFFs.

**8. Why is an API Gateway considered a critical single point of failure, and how is that risk mitigated?**
Because it sits in the path of essentially all client traffic to the system — if it goes down, clients can't reach any backend service even if those services are healthy. It's mitigated the same way as any critical component: deploying multiple redundant gateway instances behind their own load-balanced, multi-zone setup.

**9. A company has a mobile app team and a web app team frequently blocked waiting on a shared "generic API" team to add fields they each need. How might the BFF pattern help?**
Each client team could own its own BFF, tailoring the API shape and aggregation to their exact needs without waiting on a shared team to negotiate a one-size-fits-all contract, while the underlying services remain owned by their respective service teams.

**10. What's the difference in role between an API Gateway and a plain reverse proxy?**
A reverse proxy mainly hides backend servers and does routing/TLS termination at the transport/HTTP level. An API Gateway builds on that but adds application-aware, API-specific capabilities: authentication/authorization, rate limiting per client, request/response transformation, and multi-service aggregation.

**11. Why might request/response transformation at the gateway be useful in a system with a mix of legacy and modern services?**
The gateway can translate an external REST call into whatever protocol a given backend actually speaks internally (e.g., gRPC or SOAP for a legacy service), letting external clients use a consistent modern interface without every backend service needing to support it directly.

**12. In an interview, how would you decide whether a system needs a plain API Gateway, or a Gateway plus separate BFFs per client?**
If all clients need roughly the same data shape and access patterns, a single gateway (possibly with light per-route customization) is usually sufficient. If client types have meaningfully different constraints — e.g., a bandwidth-constrained mobile app versus a data-rich admin dashboard — separate BFFs behind the gateway are justified, trading some duplication for each client team's autonomy and a better-fit API for each consumer.
