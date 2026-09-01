# Notes: Scalability Basics — Vertical vs Horizontal Scaling

## Definition

**Scalability**: a system's ability to handle increasing load (users, requests, data) by adding resources, ideally without performance degradation or a full redesign.

## Vertical vs Horizontal Scaling

| Aspect | Vertical Scaling (Scale Up) | Horizontal Scaling (Scale Out) |
|---|---|---|
| Method | Add more CPU/RAM/storage to one machine | Add more machines to the pool |
| Analogy | Upgrade to a bigger truck | Add more vans to a fleet |
| Complexity | Low — app code often unchanged | Higher — needs load balancing, statelessness |
| Ceiling | Hard physical/cost limit | Practically unlimited |
| Fault tolerance | Poor — single point of failure | Good — other machines survive if one fails |
| Cost curve | Non-linear; top-tier hardware is disproportionately expensive | Often more cost-effective at scale; adds redundancy |
| Requires | Nothing extra | Load balancer, stateless services, shared data store |

## Prerequisites for Horizontal Scaling

1. **Load balancer** — distributes requests across servers (Module 2).
2. **Stateless application servers** — any server can handle any request.
3. **Shared/external data store** — database or cache accessible to all servers (Modules 3-4).

## Rule of Thumb

> "Vertical scaling buys you time; horizontal scaling buys you a future."

Common growth pattern: start vertical (simple, cheap early on) → transition to horizontal as you hit the ceiling of a single machine or need redundancy.

## Quick Revision Bullets

- Scalability = handling growth by adding resources, not rewriting the system.
- Vertical = bigger machine; simple but capped and fragile (SPOF).
- Horizontal = more machines; scalable and resilient but operationally complex.
- Horizontal scaling requires statelessness + load balancing + shared data storage.
- Real systems typically use both, transitioning from vertical to horizontal over time.
