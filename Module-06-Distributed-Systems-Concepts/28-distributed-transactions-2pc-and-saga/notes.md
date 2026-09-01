# Study Notes: Distributed Transactions — 2PC & Saga

## Definitions

- **Distributed transaction**: A logical unit of work that must appear atomic (all-or-nothing) across two or more independently-failing resources — separate databases, services, or message brokers — coordinated over an unreliable network.
- **Two-Phase Commit (2PC)**: A blocking atomic commitment protocol where a coordinator asks all participants to vote (Prepare phase), then instructs all participants to commit or abort based on the unanimous result (Commit phase).
- **Saga**: A sequence of local transactions, each committed independently in its own service/database, where a failure at any step triggers compensating transactions for every previously-completed step, executed in reverse order.
- **Compensating transaction**: A separate, deliberately-designed business operation that semantically undoes the effect of a previously committed local transaction (e.g., "refund payment" undoes "charge card"). It is not a true rollback — the original effect was already visible/committed to the world.
- **Orchestration (saga)**: A central orchestrator component explicitly invokes each service's local transaction and, on failure, explicitly invokes the necessary compensating transactions.
- **Choreography (saga)**: Each service publishes and listens to domain events, independently deciding to run its own local transaction or compensation in reaction — no central coordinator.
- **XA**: The X/Open standard interface that lets a transaction manager coordinate 2PC across multiple XA-compliant resource managers (databases, message brokers).

## Comparison Table: 2PC vs Saga

| Dimension | Two-Phase Commit (2PC) | Saga |
|---|---|---|
| Atomicity | True atomic commit — all participants commit or none do | No true atomicity — eventual consistency via compensation; a temporary "partially applied" state exists |
| Isolation | Preserved — locks held by participants until final decision | Not preserved — intermediate state from committed local transactions is visible to others |
| Availability | Reduced — coordinator crash after Prepare blocks all participants indefinitely (blocking protocol) | High — each local transaction commits and releases resources immediately; failures handled asynchronously |
| Failure handling | Participants block, holding locks, until coordinator recovers (or a heuristic/manual decision is made) | Compensating transactions run to semantically undo completed steps; must be designed per step |
| Complexity | Protocol/infrastructure complexity (needs XA-compliant resources, transaction manager); simpler application code | Application-level complexity (must design compensations, handle ordering, duplication, partial compensation failure) |
| Latency | Higher — synchronous, holds locks across a network round trip to every participant | Lower / non-blocking — local commits happen immediately; overall saga may take longer to fully settle but doesn't block |
| Best use case | Small number of tightly-coupled participants in a trusted boundary (e.g., monolith + 2 DBs, short-lived ledger ops) | Loosely-coupled microservices, independently deployed/owned services, long-running business processes |

## Saga Failure-Handling Considerations

- **Idempotent compensations**: Compensating transactions (and forward steps) must be safe to execute more than once, since retries and at-least-once delivery are common in distributed messaging.
- **Ordering guarantees**: Events/steps may arrive out of order or be duplicated; the saga logic (orchestrator or each choreography participant) must tolerate or explicitly sequence this — e.g., using saga IDs and step versioning.
- **Semantic lock**: Since isolation is lost, use an application-level "semantic lock" — e.g., marking a record as `PENDING`/`reserved` — so other transactions know not to treat in-flight saga state as final, reducing dirty-read-style anomalies.
- **Compensation failure**: A compensating transaction can itself fail (e.g., refund API is down) — needs retry with backoff, dead-letter handling, and possibly manual/ops intervention paths.
- **Non-reversible steps**: Some actions (e.g., sending a physical package, sending an email) cannot be truly compensated — only mitigated (e.g., "send apology email," "issue return label"). Design sagas so irreversible steps happen last.
- **Timeouts**: Each local transaction step should have a timeout after which the saga assumes failure and begins compensation, rather than waiting indefinitely.
- **Observability**: Because logic is spread across steps/events, sagas need strong tracing/correlation IDs to reconstruct what happened, especially for choreography-based sagas.

## Quick Interview Revision Bullets

- 2PC = strong atomicity + isolation, but blocking and availability-limited if coordinator fails after Prepare.
- Saga = no cross-service isolation/atomicity, but high availability; uses compensating transactions, not rollbacks.
- Compensating transaction ≠ rollback: it's a new, forward-moving operation that undoes effects that were already visible.
- Choreography = event-driven, decentralized, harder to trace; Orchestration = central coordinator, easier to reason about and monitor.
- XA is the standards-based mechanism most real 2PC implementations use.
- Real systems (Temporal, Camunda, AWS Step Functions) provide durable execution to implement sagas without hand-rolling state persistence.
- Rule of thumb: 2PC for a few tightly-coupled resources in one trust boundary; Saga for loosely-coupled microservices.
