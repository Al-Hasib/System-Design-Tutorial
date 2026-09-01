# Practice & Interview Questions

**1. What is a distributed transaction, and why can't a normal ACID transaction be used once a transaction spans two independent services or databases?**
A distributed transaction is a unit of work that must be atomic across two or more independently-failing resources coordinated over an unreliable network. Normal ACID transactions rely on a single transaction manager with direct, synchronous control over locks and a commit log for all involved data — usually within one database engine. Once the "transaction" crosses a service or network boundary, no single component has that shared control, so atomicity has to be coordinated explicitly via a protocol (2PC) or redesigned around eventual consistency (Saga).

**2. Walk through the two phases of Two-Phase Commit. What exactly does a participant do when it votes "yes"?**
- Phase 1 (Prepare/Vote): the coordinator asks every participant to prepare; each participant does all the work needed to commit (acquire locks, validate constraints) and durably logs a "prepared" record before replying with a vote.
- Phase 2 (Commit/Abort): if all votes are "yes," the coordinator tells everyone to commit; if any vote is "no" (or times out), it tells everyone to abort.
- Voting "yes" is a durable promise: the participant must be able to commit later no matter what, even across its own crash and restart, until it hears the final decision.

**3. Why is 2PC considered a blocking protocol, and what happens if the coordinator crashes after sending Prepare but before sending Commit?**
It's blocking because a participant that has voted "yes" cannot unilaterally decide to commit or abort — doing so risks disagreeing with what other participants were told. If the coordinator crashes after collecting votes but before broadcasting the decision, every participant is stuck in the "prepared" state, holding locks on live data, until the coordinator recovers (or a human/heuristic intervenes). This directly reduces availability, which is the core criticism of 2PC.

**4. What is XA, and how does it relate to 2PC?**
XA is the X/Open standard interface that lets a transaction manager coordinate a two-phase commit across multiple XA-compliant "resource managers" — for example, two relational databases, or a database plus a message queue. It's the mechanism behind most production 2PC implementations (e.g., Java's JTA), letting heterogeneous resources participate in one atomic distributed transaction.

**5. Define the Saga pattern and explain what a compensating transaction is. Why is a compensating transaction not the same as a rollback?**
A saga is a sequence of local transactions, each committed independently, where a failure at any step triggers compensating transactions for the already-completed steps, run in reverse order. A compensating transaction is a new, separate business operation designed to semantically undo the effect of a prior committed transaction (e.g., "refund" undoes "charge"). It differs from a rollback because the original operation was already committed and visible to the rest of the system before the compensation runs — a rollback erases uncommitted work, a compensation adds a new corrective action on top of committed work.

**6. Compare choreography-based and orchestration-based sagas. What are the tradeoffs of each?**
Choreography has each service publish/consume domain events and decide independently what local transaction or compensation to run — it's loosely coupled and has no single point of coordination failure, but it's hard to trace the overall flow since logic is scattered across every service's event handlers. Orchestration uses a central orchestrator that explicitly calls each service and explicitly invokes compensations on failure — it's easy to reason about, test, and monitor as one flow, but the orchestrator becomes critical infrastructure that everything depends on (though its failure is recoverable, unlike a 2PC coordinator's failure, since no cross-service locks are held).

**7. Design an e-commerce checkout flow spanning Order, Payment, and Inventory services using the Saga pattern. What are the compensating transactions for each step?**
- Order Service: create order in `PENDING` state → compensation: mark order `CANCELLED`.
- Payment Service: charge the customer's card → compensation: refund the charge.
- Inventory Service: reserve stock for the order → compensation: release the reserved stock back to available inventory.
If Inventory reservation fails after payment succeeded, the saga (via orchestrator or event chain) triggers "refund payment" and then "cancel order," in that reverse order, so the customer is never charged for an order that can't be fulfilled.

**8. Why do sagas sacrifice isolation, and what practical technique can mitigate the resulting anomalies?**
Because each local transaction commits immediately and independently, there is no cross-service lock preventing other transactions from observing intermediate, not-yet-finalized saga state (e.g., an order marked "confirmed" before inventory is actually reserved). A common mitigation is a "semantic lock" — marking the affected record with a pending/reserved status so other parts of the system know to treat it as provisional rather than final until the saga completes or compensates.

**9. Why must saga steps and compensating transactions be idempotent?**
Distributed messaging and retries commonly provide at-least-once delivery, so the same command or event can be processed more than once (e.g., due to a timeout-triggered retry). If "charge card" or "refund card" isn't idempotent, a retried message could double-charge or double-refund the customer. Idempotency (e.g., via unique request/operation IDs checked before applying an effect) ensures repeated delivery doesn't corrupt state.

**10. In what scenario would you actually choose 2PC over a Saga in a modern system?**
2PC makes sense when you have a small number of tightly-coupled resources within a single trust/infrastructure boundary and you genuinely cannot tolerate intermediate, partially-applied state being visible — for example, a monolith writing atomically to two databases, or coordinating a database write with a message enqueue via an XA-compliant broker for exactly-once-like delivery. It's a poor fit across independently deployed microservices because it couples their availability together and blocks on coordinator failure.

**11. What role do tools like Temporal or Camunda play in implementing sagas, and why do teams prefer them over hand-rolled orchestration?**
They provide durable execution: the engine persists the state of a long-running workflow after every step, so if a worker process crashes, execution resumes exactly where it left off rather than losing state or needing to be manually reconciled. They also provide built-in constructs for defining compensating actions tied to each forward step, automatic retries, and visibility/monitoring into in-flight sagas — replacing a lot of the error-prone bookkeeping teams would otherwise have to build themselves for orchestration-based sagas.

**12. A junior engineer proposes using 2PC across five microservices owned by five different teams to "guarantee consistency." What would you push back on?**
2PC would couple the availability of all five services together — if any one participant is slow, unreachable, or its coordinator crashes mid-protocol, every other participant is blocked holding locks, even though those services are otherwise healthy and independently deployed. It also requires all five to support a shared transaction coordination mechanism (e.g., XA), which is unusual for typical HTTP/event-based microservices. A saga would be the more appropriate choice: each service keeps its independent availability, and cross-service consistency is achieved eventually through compensating transactions instead of synchronous locking.
