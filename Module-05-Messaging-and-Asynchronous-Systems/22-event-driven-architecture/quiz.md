# Practice & Interview Questions

**1. How does event-driven architecture differ from a traditional request-driven (synchronous API call) architecture?**
In request-driven architecture, a service calls another service's API and blocks/waits for a response, creating tight temporal coupling. In event-driven architecture, a service publishes an event describing a fact ("X happened") and moves on; any number of other services react independently and asynchronously, with no direct dependency between producer and consumer availability.

**2. Why are events typically named in the past tense (e.g., "OrderPlaced" rather than "PlaceOrder")?**
Because an event represents an immutable fact about something that has already occurred, not a request or command for something to be done. "PlaceOrder" implies an instruction the receiver must obey; "OrderPlaced" simply states a fact that listeners may optionally react to, which is the core semantic distinction between commands and events.

**3. Explain the difference between event notification and event-carried state transfer.**
Event notification sends a minimal event (often just an ID) and expects the consumer to call back to the source service's API for full details, which keeps events small but preserves some runtime coupling to the producer's availability. Event-carried state transfer embeds all the data a consumer needs directly in the event payload, eliminating the callback dependency at the cost of larger, potentially duplicated data.

**4. What is event sourcing, and what's one major benefit and one major cost of using it?**
Event sourcing stores the full sequence of events that led to an entity's current state (like a ledger) rather than just the current state itself, deriving the current state by replaying those events. Benefit: complete audit trail and the ability to reconstruct state at any point in time. Cost: significantly more complexity for querying current state efficiently and for tooling (often requiring CQRS and snapshotting).

**5. What is "eventual consistency" and why is it an inherent property of event-driven systems?**
Eventual consistency means that after an event is published, there is a window of time during which different parts of the system may hold different, temporarily inconsistent views of the same underlying data, because consumers process events asynchronously and at their own pace. It's inherent to EDA because decoupling producers and consumers in time necessarily removes the guarantee that all effects of an event happen instantaneously and atomically everywhere.

**6. Scenario: A customer support agent looks up an order immediately after a customer places it and sees "processing" instead of "confirmed," even though the payment actually succeeded a moment ago. What's happening, and how would you explain this to a non-technical stakeholder?**
This is the eventual consistency window in action: the payment confirmation event was published but the order-status service hasn't yet processed it and updated its view. You'd explain that different parts of the system update independently within a very short delay (often milliseconds), and the UI could show a "processing" indicator or add a brief polling/refresh mechanism to bridge that gap for the user.

**7. Why is debugging harder in an event-driven system compared to a synchronous, request-driven one, and what tools mitigate this?**
In a synchronous system, a single call stack shows the entire flow of execution, making cause and effect easy to trace. In EDA, behavior emerges from many independently deployed producers and consumers reacting to events, so there's no single call stack to follow. Distributed tracing with correlation/causation IDs, centralized event logging, and well-documented event schemas mitigate this by letting you reconstruct the causal chain across services after the fact.

**8. How does event-driven architecture support independent team autonomy in a large organization?**
Because producers don't need to know who consumes their events, new teams can build features that subscribe to existing events without requiring any code changes, coordination, or deployment from the producing team. This lets teams add functionality, experiment, and deploy independently, which is much harder in a request-driven system where the caller must explicitly integrate with every downstream service.

**9. What risk does "event-carried state transfer" introduce regarding data freshness, and how might you address it?**
Because the event snapshot captured at publish time might grow stale if the consumer stores it and the source data later changes, consumers relying purely on event payloads can end up acting on outdated data. This is typically addressed by also including update/versioned events whenever the underlying entity changes, or by treating event data as a "was true at this point in time" fact rather than a live source of truth for anything requiring strict currency.

**10. When would you deliberately avoid event-driven architecture in favor of a simple synchronous request?**
When the caller genuinely needs an immediate, guaranteed result before proceeding — for example, checking a payment authorization before allowing checkout to complete — a synchronous call is simpler to reason about and avoids the complexity of eventual consistency. EDA is best reserved for side effects, notifications, and workflows that don't need to block the primary user-facing operation.

**11. What is a "correlation ID" and why is it important in event-driven systems?**
A correlation ID is a unique identifier attached to an initial request or event and propagated through every subsequent event and service call that stems from it. It's important because it lets engineers reconstruct and trace the entire causal chain of a business process across many independently-reacting services, which is otherwise very difficult in an asynchronous, decoupled architecture.

**12. Compare the coupling risk of event notification vs event sourcing from a schema-evolution perspective.**
Event notification events are small (often just an ID and type), so schema changes are low-risk since there's little payload to evolve — but consumers remain coupled to the producer's API contract for details. Event sourcing events are the permanent source of truth, so their schema must be extremely carefully versioned since you may need to replay old events years later — a poorly planned schema change can break the ability to reconstruct historical state correctly.
