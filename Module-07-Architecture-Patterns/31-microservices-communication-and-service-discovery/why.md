# Why This Topic Matters: Microservices Communication & Service Discovery

> **In one sentence:** Splitting a system into services immediately creates two problems that didn't exist before — how does a service find another one whose instances are constantly changing, and what happens to the caller when that other one is slow or broken.

## The World Before This Idea

You've split into services. Now the order service needs to call the inventory service. Where is it?

The instinctive answer is a config file with an IP address. That works until the inventory service autoscales from 3 instances to 12, and 9 of them get no traffic. Or it gets redeployed onto new hosts with new IPs, and the config is stale. Or one instance dies and callers keep sending requests to a dead address until someone notices.

So you put a load balancer in front and hardcode its address. Better — but now you need a load balancer per service, each one manually configured, each one a thing to provision whenever a new service appears. In a Kubernetes cluster where pods are created and destroyed continuously, manual configuration is not a strategy.

## The Problems It Solves

### 1. Addresses that change constantly
**What you see:** Calls to dead instances, uneven load across a fleet, and deploys that require updating configuration in several other services.

**Why it happens:** In an elastic, containerized environment, instance addresses are ephemeral by design. Any static mapping is stale the moment it's written.

**How service discovery solves it:** Instances register themselves in a registry (Consul, etcd, ZooKeeper, or Kubernetes' built-in service objects) on startup and deregister on shutdown. Callers look up "inventory-service" by name and get the current healthy set. Health checks evict dead instances automatically. The address problem stops being a human problem.

### 2. Choosing the wrong communication style
**What you see:** A checkout flow that makes nine synchronous calls and takes four seconds, or an inventory update sent asynchronously when the caller genuinely needed to know whether stock was reserved.

**Why it happens:** Teams pick one style — usually synchronous REST, because it's familiar — and apply it everywhere.

**How this topic solves it:** It separates the cases. **Synchronous** (REST, gRPC) when the caller needs the answer to proceed: pricing, authorization, stock checks. **Asynchronous** (queues, events) when the caller just needs the work to happen eventually: notifications, indexing, analytics, downstream reactions. Getting this split right is most of the difference between a responsive system and a fragile one, because every synchronous call adds its latency and subtracts from your availability.

### 3. Chatty calls that multiply latency
**What you see:** Rendering one page triggers 200 internal calls — the distributed version of the N+1 query problem.

**Why it happens:** Service boundaries drawn without regard to access patterns force callers to loop over remote calls that used to be a local join.

**How this topic solves it:** It makes call-graph depth and fan-out a design concern. Remedies include batch endpoints, aggregation at a BFF, data duplication across service boundaries, and — most importantly — reconsidering boundaries that require a loop of remote calls to satisfy one screen.

### 4. Protocol overhead at internal volumes
**What you see:** Services spending meaningful CPU on JSON serialization and HTTP header parsing for millions of small internal calls.

**Why it happens:** JSON over HTTP/1.1 is excellent for public APIs — human-readable, universally supported, debuggable — and comparatively expensive for high-volume internal traffic.

**How gRPC solves it:** Binary Protocol Buffers over HTTP/2 gives smaller payloads, faster serialization, multiplexed streams over one connection, and a generated, type-checked contract on both sides. The cost is losing human readability and easy `curl`-ability, which is why the common pattern is REST at the edge, gRPC inside.

## The Price You Pay

- **The registry is critical infrastructure.** If service discovery is down, nothing can find anything. It needs to be highly available (which is why it's usually consensus-backed) and clients need to cache the last known good set and degrade gracefully.
- **Client-side vs server-side discovery is a real trade.** Client-side (the caller queries the registry and load balances itself) is efficient and removes a hop, but puts discovery logic in every service and every language. Server-side (a proxy does it) centralizes the logic at the cost of an extra hop. Service meshes resolve this with a sidecar proxy — and add a whole control plane to operate.
- **Registration can lie.** An instance that registers before it's actually ready receives traffic it can't serve; one that crashes without deregistering leaves a ghost entry until health checks catch it. Readiness versus liveness distinctions exist for exactly this reason.
- **Every synchronous call is a coupling.** It transmits latency and failure upward. This is why topic 26's patterns are not optional here.
- **Versioning is now a distributed contract problem.** Changing a request or response shape can break callers you don't control, so you need backward-compatible evolution and a deprecation process.

## When You Need It — and When You Don't

| You need discovery when | You can skip it when |
|---|---|
| Instances are ephemeral (containers, autoscaling) | A handful of services on fixed hosts |
| Services are added and removed frequently | DNS plus a static load balancer is sufficient |
| You run on Kubernetes or a cloud orchestrator | A monolith with in-process calls |
| You need health-aware routing and failover | — |

| Synchronous when | Asynchronous when |
|---|---|
| The caller needs the result to continue | The work can complete later |
| The operation is fast and reliable | The downstream is slow, bursty, or flaky |
| Strong consistency is required | Multiple consumers need the same event |

## Why This Shows Up in Interviews

Once you've drawn a microservices diagram, "how do these services find each other?" and "what happens if this one is down?" follow almost automatically. Interviewers want to hear service discovery named (and, in a Kubernetes context, that it's built in), a deliberate synchronous/asynchronous split with reasons, and resilience patterns applied to every synchronous hop. A candidate who says "I'd use gRPC internally for the high-volume calls and keep REST at the edge for the public API, with events for anything the caller doesn't need to wait on" is demonstrating exactly the right level of thinking.

## How It Connects

This is the operational reality of **microservices** (topic 30). The registry typically runs on a **consensus**-backed store (topic 27). Synchronous calls require **circuit breakers and retries** (topic 26); asynchronous ones require **queues** (topic 20), **Pub/Sub** (topic 21), and **idempotency** (topic 29). Client-side load balancing often uses **consistent hashing** (topic 24). The **API gateway** (topic 9) is the external face of all this, **gRPC and Protocol Buffers** are covered in topics 33 and 35, and **Kubernetes** (topic 44) provides discovery natively.

**Next:** [Domain-Driven Design Basics](../32-domain-driven-design-basics/why.md) — how to decide where the service boundaries should have been in the first place.
