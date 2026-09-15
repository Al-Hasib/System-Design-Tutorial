# Why This Topic Matters: Distributed Transactions (2PC & Saga)

> **In one sentence:** A local database transaction gives you all-or-nothing for free inside one database — and the moment your operation spans two databases or two services, that guarantee vanishes and you have to rebuild it by hand.

## The World Before This Idea

An order requires three things: charge the payment service, decrement inventory, create a shipment. Three services, three databases.

You call them in sequence. Payment succeeds. Inventory succeeds. Shipment fails — the service is down. Now what?

The customer has been charged for an order that will never ship, and inventory shows one fewer item that nobody has. Your code has no rollback: you cannot un-call an HTTP request. So you write a compensating call to refund the payment — and that call fails too, because the payment service just started having problems as well. Now the money is gone and nobody knows.

Multiply this across every multi-service operation. Reconciliation scripts, manual support tickets, and quiet money leakage are the normal end state for systems that never addressed this.

## The Problems It Solves

### 1. Partial completion with no way back
**What you see:** Orders in impossible states. Money moved without goods. Inventory decremented for cancelled orders.

**Why it happens:** Each service commits independently. There is no shared transaction to roll back, and a failure after step N leaves steps 1 through N-1 permanently applied.

**How 2PC solves it:** A coordinator asks every participant whether it can commit, in a **prepare** phase. Each participant does the work but holds it uncommitted, and promises it can commit. Only if everyone agrees does the coordinator send **commit**. If anyone refuses, everyone aborts. This gives genuine atomicity across systems.

**How Saga solves it:** Accept that each step commits immediately, and define an explicit **compensating action** for each one — refund reverses charge, restock reverses decrement. If step three fails, run the compensations for steps two and one in reverse. The result is not atomic (there is an observable window where the charge happened and the refund has not) but it is eventually correct, and it holds no locks.

### 2. Locks held across the network
**What you see:** With 2PC in production, a coordinator crash leaves participants holding locks indefinitely. Rows are frozen, throughput collapses, and the system is effectively down until someone intervenes.

**Why it happens:** Between prepare and commit, participants must hold resources locked — they have promised they can commit, so they cannot let anyone else change that data. If the coordinator never sends the decision, participants are stuck. This is the famous **blocking** problem of 2PC, and it is why 2PC is rare in modern microservice architectures despite being theoretically cleaner.

**How Saga solves it:** No distributed locks at all. Each local transaction commits and releases immediately. This is the core reason sagas dominate in practice: they trade atomicity for availability and throughput, which is usually the right trade at scale.

### 3. Compensations that cannot actually undo
**What you see:** You need to un-send an email, un-launch a rocket, or restore a seat that someone else has already booked.

**Why it happens:** Not every action is reversible. Compensation is a business-level concept, not a technical one.

**How this topic solves it:** It forces you to design the compensation together with the action, and to order steps so irreversible ones come last. "Reserve the seat, charge the card, then send the confirmation email" is a saga that works; putting the email first is one that does not. This sequencing discipline is most of the practical skill.

### 4. Choreography that nobody can follow
**What you see:** A saga implemented as services reacting to each other's events, and after twelve steps no one can explain the flow, debug a stuck instance, or say which compensations have run.

**Why it happens:** Choreographed sagas — each service listens and reacts — are loosely coupled but have no central record of the workflow.

**How orchestration solves it:** A dedicated orchestrator holds the state machine, calls each step, and drives compensations. You give up some decoupling and gain something valuable: a single place that knows where every saga instance is, which is the difference between a debuggable system and an unexplainable one. For anything beyond three or four steps, orchestration is usually worth it.

## The Price You Pay

- **2PC costs:** blocking on coordinator failure, poor throughput from cross-network locks, latency of two round trips to every participant, and a coordinator that must itself be highly available (usually via consensus). It is still used *within* systems — Kafka transactions, XA in some enterprise stacks, distributed databases internally — but it is a poor fit across independently operated services.
- **Saga costs:** no isolation at all. Intermediate states are visible to other transactions, so a concurrent reader can see money debited before it is credited. Preventing the anomalies this causes requires application-level techniques such as semantic locks, pessimistic reads, or re-reading values before compensating.
- **Compensations double the code.** Every step needs a tested reverse. Compensations must be idempotent and must themselves be retried until they succeed — a failed compensation is the worst state in the system.
- **Debugging spans services and time.** A saga stuck at step four, an hour ago, with two compensations pending, is genuinely hard to reason about without strong tracing and a persisted state machine.

**And the best option is often neither.** If two pieces of data must change atomically, the strongest move is frequently to put them in the same service and the same database and use a local transaction. Service boundaries drawn so that transactions must span them are usually boundaries drawn in the wrong place.

## When You Need It — and When You Don't

| Use 2PC when | Use Saga when | Use neither when |
|---|---|---|
| Participants share one trust and ops boundary | Steps span independently operated services | The data can live in one database |
| Strict atomicity is required | Workflows are long-running (seconds to days) | You can redraw the service boundary |
| Volume is low and latency-tolerant | High throughput and availability matter | The operation is naturally single-service |
| — | Compensating actions are definable | — |

## Why This Shows Up in Interviews

Any design with microservices and a multi-step business process — checkout, booking, ride matching, money transfer — runs into this. Interviewers ask what happens when step three fails, and they are checking whether you know that cross-service atomicity is not available for free. The expected answer names the saga pattern, defines the compensations concretely, mentions orchestration versus choreography, and acknowledges the lost isolation. Strong candidates add the framing above: first ask whether the boundary should exist at all.

## How It Connects

This problem is created by **microservices** (topic 30) and by **sharding** (topic 14), both of which break the single-database transaction. It is the practical fallout of **ACID vs BASE** (topic 16) and of **CAP** (topic 15). Sagas are implemented over **message queues** (topic 20) and **event-driven** flows (topic 22), and they absolutely require **idempotency** (topic 29), since every step and compensation will be retried. The 2PC coordinator typically relies on **consensus** (topic 27) to be highly available, and **domain-driven design** (topic 32) is the discipline for drawing boundaries that minimize the need for any of this.

**Next:** [Data Consistency Models & Idempotency](../29-data-consistency-models-and-idempotency/why.md) — the guarantees and the safety net underneath all of it.
