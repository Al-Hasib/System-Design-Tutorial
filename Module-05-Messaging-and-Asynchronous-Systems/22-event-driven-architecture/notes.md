# Study Notes: Event-Driven Architecture

## Core Definitions

- **Event-Driven Architecture (EDA)**: an architectural style where system components communicate primarily by producing and consuming events (facts about things that happened) rather than through direct, synchronous calls.
- **Event**: an immutable record that something occurred, typically named in the past tense (e.g., `OrderPlaced`, `PaymentProcessed`).
- **Event producer**: emits events when something notable happens within its domain.
- **Event consumer**: subscribes to and reacts to relevant events.
- **Event broker / event bus**: the infrastructure (Kafka, RabbitMQ, cloud pub-sub, etc.) that transports events from producers to consumers.

## Request-Driven vs Event-Driven

| Aspect | Request-Driven | Event-Driven |
|---|---|---|
| Communication | Synchronous call, waits for response | Asynchronous, fire-and-react |
| Coupling | Tight temporal coupling (both must be up at once) | Loose coupling in time, space, and control flow |
| Control flow | Explicit, linear, easy to trace (call stack) | Emergent, distributed across producers/consumers |
| Failure impact | Caller blocked/fails if callee is down | Producer unaffected; consumer catches up later |
| Adding new functionality | Often requires modifying the caller | New consumer just subscribes; no upstream changes |

## Three Flavors of Events

| Style | Description | Tradeoff |
|---|---|---|
| Event notification | Thin event, minimal data (e.g., just an ID); consumer calls back to source for details | Small events, but reintroduces coupling/availability dependency on producer's API |
| Event-carried state transfer | Event includes all data the consumer needs | Maximizes decoupling/resilience; larger payloads, must manage data freshness |
| Event sourcing | Entire history of events is the source of truth; current state = replay of events | Full audit trail, time-travel/rebuild capability; significant query/tooling complexity |

## Benefits of EDA

- Independent scaling of producers and consumers.
- Resilience: a failing/slow consumer doesn't affect the producer or other consumers.
- Extensibility: new features subscribe to existing events without modifying upstream services.
- Natural fit for organizations with many autonomous teams/services.

## Challenges of EDA

- **Eventual consistency**: different parts of the system may briefly hold different views of the same data.
- **Debuggability/complexity**: behavior emerges from a web of producers/consumers rather than a single call stack; requires distributed tracing, correlation IDs, and well-defined event schemas.
- **Schema evolution**: changing an event's shape can break consumers that depend on old fields.
- **Testing complexity**: harder to test end-to-end flows that span multiple independently-deployed reactive services.

## Quick Summary

- EDA replaces "call and wait" with "announce and react."
- Choose event style based on how much data consumers need and how much coupling to the producer's API you can tolerate.
- Event sourcing is powerful but heavyweight — reserve it for domains where audit trail/time-travel is genuinely valuable (e.g., financial ledgers).
- Invest early in observability (tracing, correlation IDs, event schemas/versioning) — EDA complexity without these tools becomes unmanageable at scale.
