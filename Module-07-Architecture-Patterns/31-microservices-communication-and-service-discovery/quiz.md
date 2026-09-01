# Practice & Interview Questions

**1. When should a service call another service synchronously versus asynchronously?**
Use synchronous calls when the caller needs an immediate answer to continue its current operation (e.g., checking real-time inventory before confirming an order). Use asynchronous events when the receiver is just reacting to something that already happened and the caller doesn't need to block (e.g., sending a confirmation email, updating analytics).

**2. Why can't microservices just use hardcoded IP addresses for each other?**
In modern deployments, instances scale up/down dynamically, get replaced on failure, and receive new IPs on every restart (especially in container orchestrators like Kubernetes). A hardcoded address would go stale almost immediately, so services need a way to look up current, healthy addresses at runtime — service discovery.

**3. Compare client-side and server-side service discovery.**
In client-side discovery, the calling service itself queries the service registry and chooses which healthy instance to call (e.g., Netflix Eureka + Ribbon). In server-side discovery, the client calls a stable endpoint (a load balancer or platform abstraction), and that intermediary queries the registry and forwards the request (e.g., Kubernetes Services). Client-side gives the caller more control; server-side keeps client code simpler at the cost of an extra hop.

**4. What is a service mesh, and what problem does it solve beyond basic service discovery?**
A service mesh (e.g., Istio, Linkerd) deploys a sidecar proxy alongside every service instance to transparently handle discovery, load balancing, retries/timeouts, mutual TLS encryption, and observability — all outside application code. It solves the problem of reimplementing these cross-cutting concerns in every service/language by centralizing them in infrastructure.

**5. Describe the lifecycle of an instance in a service registry.**
On startup, the instance registers itself with its address and a health check endpoint. The registry periodically polls that health check; healthy instances stay listed and are returned to callers, unhealthy ones are excluded. On graceful shutdown the instance deregisters itself; on a crash, failed health checks eventually remove it.

**6. In an interview, you're designing an e-commerce checkout flow. Which parts should be synchronous and which asynchronous?**
Synchronous: validating payment authorization and confirming inventory availability, since the user is waiting for a definite yes/no to complete the purchase. Asynchronous: sending the order confirmation email, updating recommendation/analytics systems, and notifying the warehouse for fulfillment — none of these need to block the checkout response.

**7. What's the main downside of asynchronous, event-driven communication compared to synchronous calls?**
It introduces eventual consistency — other services may not reflect a change immediately, and reasoning about the overall system state at any given instant becomes harder. It also adds infrastructure complexity (message broker, ordering guarantees, dead-letter handling).

**8. Why is a load balancer alone (as covered in Module 2) not sufficient for microservices without service discovery?**
A traditional load balancer typically needs its backend pool configured, which is static or slowly updated. Service discovery makes that pool dynamic and self-updating in real time as instances register, deregister, and pass/fail health checks — server-side discovery is essentially a load balancer wired directly into a live service registry.

**9. What is Netflix Eureka and what discovery pattern does it represent?**
Eureka is Netflix's open-source service registry, historically paired with the Ribbon client-side load balancer. It represents the client-side discovery pattern: services query Eureka directly for a list of healthy instances and pick one themselves.

**10. If a downstream service becomes slow (not fully down), how does that interact with synchronous service-to-service calls, and what pattern from Module 6 helps?**
A slow downstream service can cause the caller to block, tie up resources (threads/connections), and cascade slowness upstream. Circuit breakers, timeouts, and bulkheads (Module 6) help by failing fast, capping wait time, and isolating resource pools so one slow dependency doesn't take down the whole calling service.

**11. What role does an API gateway (Module 2) play alongside service discovery?**
The API gateway is typically the external-facing entry point that routes client requests to the right internal service, often using server-side discovery internally to find healthy backend instances. It also centralizes concerns like authentication, rate limiting, and request routing, while service discovery specifically handles finding live instances of each internal service.

**12. Why might a team choose Kubernetes' built-in service discovery over adopting a full service mesh?**
Kubernetes Services + DNS + kube-proxy already provide solid server-side discovery and load balancing out of the box with minimal extra operational burden. A full service mesh adds significant complexity (control plane, sidecar injection, certificate management) that's only worth it when you need its extra features — fine-grained traffic control, mutual TLS everywhere, rich observability — across a large number of services.
