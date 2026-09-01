# Notes: Microservices Communication & Service Discovery

## Definitions

- **Synchronous communication**: Request/response calls (REST, gRPC) where the caller blocks until it gets a reply.
- **Asynchronous communication**: Event/message-based communication (via a broker/queue like Kafka or RabbitMQ) where the caller does not block on the receiver processing the message.
- **Service discovery**: The mechanism that lets a service find the network location of healthy instances of another service at runtime, without hardcoded addresses.
- **Service registry**: A database of currently available service instances and their health status (e.g., Consul, Eureka, etcd, Kubernetes API server).
- **Service mesh**: Infrastructure layer (e.g., Istio, Linkerd) using per-instance sidecar proxies (often Envoy) to handle discovery, load balancing, retries, mTLS, and observability transparently.
- **Sidecar proxy**: A proxy process deployed alongside each service instance that intercepts all its network traffic.

## Communication Styles Comparison

| Aspect | Synchronous (REST/gRPC) | Asynchronous (Queue/Event) |
|---|---|---|
| Caller blocks? | Yes, waits for response | No, fire-and-forget or event publish |
| Coupling | Tighter — callee's availability affects caller | Looser — decoupled in time |
| Use case | Need an immediate answer (e.g., check stock) | Reacting to something that happened (e.g., send email) |
| Failure handling | Needs timeouts/retries/circuit breakers | Broker buffers messages; consumer catches up later |
| Consistency | Easier to reason about immediate state | Eventual consistency across services |

## Service Discovery Patterns Comparison

| Pattern | Who does the lookup | Example tech | Pros | Cons |
|---|---|---|---|---|
| Client-side discovery | The calling service queries the registry and picks an instance | Netflix Eureka + Ribbon | Full client control over load balancing | Discovery logic duplicated in every client/language |
| Server-side discovery | A load balancer/router queries the registry on the client's behalf | Kubernetes Services, AWS ELB | Clients stay simple, no discovery logic needed | Extra network hop; LB is critical infra |
| Service mesh | Sidecar proxies handle discovery transparently | Istio, Linkerd (Envoy sidecars) | Centralizes discovery, LB, retries, security, observability | High operational complexity to run the mesh itself |

## Service Registry Lifecycle

1. **Register** — new instance announces itself (address + health check endpoint) to the registry on startup.
2. **Health check** — registry polls the instance's health endpoint periodically.
3. **Serve lookups** — registry returns only healthy instance addresses to callers.
4. **Deregister** — instance removes itself on graceful shutdown, or is marked down after failed health checks on crash.

## Bullet Summary

- Choose sync vs async based on whether the caller needs an immediate answer or is just reacting to a past event.
- Service discovery solves the problem of dynamic, ephemeral instance addresses in autoscaled/containerized environments.
- Three discovery patterns: client-side, server-side, and service-mesh (sidecar-based) — differ in where the discovery/load-balancing logic lives.
- A service registry continuously tracks instance health via registration and health checks so only live instances are returned.
- Real systems combine both communication styles and often layer a service mesh on top of platform-native discovery (e.g., Kubernetes) for advanced traffic control.
