# Distributed Transactions: Two-Phase Commit & Saga Pattern

**Difficulty:** Advanced | **Estimated length:** 18-22 min | **Prerequisites:** [Consensus Algorithms: Paxos & Raft](../27-consensus-algorithms-paxos-and-raft/README.md), [ACID vs BASE, Normalization vs Denormalization](../../Module-03-Databases-and-Storage/16-acid-vs-base-normalization-vs-denormalization/README.md), [Event-Driven Architecture](../../Module-05-Messaging-and-Asynchronous-Systems/22-event-driven-architecture/README.md)

## Learning Objectives

- Explain why ACID transactions break down once a transaction spans multiple services or databases
- Describe the Two-Phase Commit (2PC) protocol in detail, including coordinator/participant roles, the Prepare and Commit phases, and its blocking failure mode
- Explain the Saga pattern, the difference between choreography-based and orchestration-based sagas, and why compensating transactions are semantic undos, not rollbacks
- Compare 2PC and Saga across consistency, availability, complexity, and latency, and know which to reach for in a real architecture
- Recognize real-world tools and patterns (XA, Temporal, Camunda, Step Functions) used to implement these approaches

## Script

### Hook / Intro

Imagine you're booking a trip: a flight, a hotel, and a rental car. You want all three or none of them — you don't want to end up with a hotel room in Paris and no flight to get there. That's the essence of a distributed transaction problem, and it's one of the gnarliest topics in distributed systems.

In a single database, you'd wrap this in a `BEGIN TRANSACTION ... COMMIT` and rely on ACID guarantees. But what happens when "the flight," "the hotel," and "the car" are three different services, each with its own database, possibly owned by three different teams, maybe even three different companies? You can't just draw a transaction boundary around all of them and expect the database engine to save you. Today we're covering the two classic answers to this problem: Two-Phase Commit, and the Saga pattern. By the end of this video you'll know exactly how each one works, where each one breaks, and which one you should actually reach for.

### Why Distributed Transactions Are Hard

Let's start with why this is hard in the first place. ACID — atomicity, consistency, isolation, durability — is a beautiful guarantee, but it's built on an assumption: that a single transaction manager has full, synchronous control over all the data involved, usually within one database process, sometimes with a fast local network connecting shards. It can lock rows, buffer writes, and either commit everything to disk atomically or throw all of it away.

Once your transaction spans a service boundary — Order Service, Payment Service, Inventory Service, each with its own datastore — you lose that shared context. Each of those local databases can still give you local ACID transactions, but nobody has a single log or lock table spanning all three. You now have to coordinate atomicity yourself, over a network, which is unreliable. A request can be delayed, dropped, duplicated, or the machine handling it can crash halfway through. And crucially, you can't tell the difference between "the remote service is slow" and "the remote service is dead" — that's the fundamental uncertainty at the heart of every distributed transaction protocol we're about to discuss.

So we need a protocol that says: here's how multiple independent parties, over an unreliable network, agree on whether a multi-step operation happened entirely or not at all — or, alternatively, a strategy that gives up on "all or nothing" and replaces it with something more forgiving.

### Two-Phase Commit (2PC)

Two-Phase Commit is the classic, textbook answer, and it tries to preserve atomicity almost exactly like a local transaction would. There are two roles: a **coordinator** — sometimes called the transaction manager — and one or more **participants**, sometimes called resource managers. Each participant is typically a database or service capable of taking part in a distributed transaction.

**Phase 1: Prepare / Vote.** The coordinator sends a "prepare" message to every participant, essentially asking, "can you commit this?" Each participant does everything short of actually committing — it acquires the necessary locks, writes the change to a durable but not-yet-visible log, checks constraints — and then replies with a vote: "yes, I can commit," or "no, abort." Critically, once a participant votes yes, it has made a promise: it must be able to commit later, no matter what, even if it crashes and restarts in the meantime. That's why the prepared state has to be written durably before the vote is sent.

**Phase 2: Commit / Abort.** The coordinator collects all the votes. If every single participant voted yes, it sends "commit" to everyone. If even one voted no — or didn't respond in time — the coordinator sends "abort" to everyone, and all participants roll back the work they tentatively staged in phase one. This is the atomicity guarantee: unanimous yes or nothing happens.

Now here's the famous problem: 2PC is a **blocking protocol**. Suppose every participant votes yes, and then the coordinator crashes before it sends the commit message. Every participant is now sitting in "prepared" state, holding locks, unable to unilaterally decide to commit or abort — because for all they know, the coordinator might have already told some other participant to commit, and if they abort on their own, the transaction becomes inconsistent. They're stuck, waiting, holding locks on live data, until the coordinator recovers or a human intervenes. That's a direct availability cost for the sake of atomicity — this is a real-world instance of the tradeoffs we discussed with Paxos and Raft: 2PC is not fault-tolerant to coordinator failure the way a consensus protocol is, because there is no quorum, no election — it's a single coordinator with a single point of blocking failure. There are extensions like Three-Phase Commit that try to fix this with timeouts, but they're rarely used in practice because they trade blocking for other complications and still can't fully solve it under network partitions.

You've likely encountered 2PC in the real world as **XA transactions** — the X/Open XA specification is a standard interface that lets a transaction manager coordinate multiple XA-compliant resources, like two different relational databases, or a database and a message queue, under one atomic transaction. Databases like PostgreSQL, MySQL, and Oracle support XA, and Java's JTA (Java Transaction API) is built around this model. It works, but it's heavyweight, and it tightly couples the availability of your entire transaction to the availability and responsiveness of every single participant plus the coordinator.

### The Saga Pattern

The Saga pattern takes a completely different philosophy: instead of trying to make a distributed operation atomic, it breaks it into a sequence of independent **local transactions**, each of which commits immediately in its own service. If a later step fails, instead of rolling back the whole chain the way a database would, the saga runs **compensating transactions** for each step that already succeeded, in reverse order, to semantically undo them.

Back to our travel-booking analogy: book the flight — that's a real, committed transaction, you actually have a ticket. Book the hotel — also committed. Now the rental car booking fails, say, no cars available. A saga doesn't "roll back" the flight booking the way a database undoes an uncommitted write — there's no undo log because it was never uncommitted. Instead, it triggers a compensating action: cancel the flight, cancel the hotel. That's a business operation — potentially with cancellation fees, refund logic, and its own failure modes — not a low-level database rollback.

There are two ways to coordinate a saga. **Choreography-based sagas** have no central coordinator: each service does its local transaction and then publishes an event, and other services react to that event by doing their own local transaction and publishing their own event. Order Service creates an order and emits `OrderCreated`. Payment Service listens for that, charges the card, and emits `PaymentCompleted` or `PaymentFailed`. Inventory Service listens for `PaymentCompleted` and reserves stock. If Inventory Service emits `InventoryReservationFailed`, Payment Service listens for that and issues a refund — its compensating transaction. This is loosely coupled and works naturally with the event-driven architecture we covered previously, but it can get hard to reason about as the number of services grows — the "transaction" logic is smeared across every participant's event handlers, and there's no single place to look to understand the whole flow.

**Orchestration-based sagas** introduce a central orchestrator — a piece of code or a workflow engine — that explicitly tells each service what to do next and explicitly invokes compensating transactions on failure. The orchestrator calls Order Service, then calls Payment Service, then calls Inventory Service, tracking state along the way. If Inventory fails, the orchestrator explicitly calls Payment's "refund" endpoint and Order's "cancel" endpoint. This centralizes the logic, making it much easier to test, monitor, and reason about, at the cost of that orchestrator becoming a critical piece of infrastructure — though notably, if the orchestrator goes down, the already-completed local transactions are still valid and committed; you "just" need to resume or recover the orchestration state, which is a fundamentally easier failure mode than 2PC's blocking problem, because no locks are being held across services while you do it.

It's essential to internalize that saga compensations are **not** true rollbacks. A true rollback pretends the original operation never happened, atomically, invisible to anyone else. A compensating transaction is a new, separate operation that happened *after* the original one was already visible to the world — other transactions might have already read that intermediate state. This means sagas sacrifice **isolation**: there's a window where partial, uncommitted-from-a-business-perspective state is visible to other parts of the system. Handling that well is genuinely hard design work, which we'll get into with the takeaways and quiz.

### 2PC vs Saga — Tradeoffs

Let's lay the tradeoffs out directly. **Atomicity**: 2PC gives you real atomicity — all participants commit or none do, guaranteed by protocol. Saga gives you eventual consistency — the sequence eventually reaches either "all committed" or "all compensated," but there's a window in between where the world is in a partial state.

**Isolation**: 2PC preserves isolation because locks are held across the entire transaction until the final decision. Saga fundamentally cannot give you isolation across steps — intermediate state is visible, which is why you often need patterns like semantic locks or versioning to avoid anomalies.

**Availability**: this is the big one. 2PC sacrifices availability — a crashed coordinator blocks every participant holding a lock. Saga preserves availability — each local transaction commits and moves on independently; a failure downstream is handled asynchronously via compensation rather than by blocking everyone upstream.

**Complexity**: 2PC's complexity lives in the protocol and infrastructure — you need XA-compliant resources and a transaction manager, but the application code is relatively declarative. Saga pushes complexity into your application logic — you must design and implement a compensating transaction for every step, handle out-of-order or duplicate events, and handle partial failure of the compensations themselves.

**Latency**: 2PC is synchronous and holds locks across a network round trip to every participant, which is slow and doesn't scale well with more participants or higher latency links. Sagas are typically asynchronous and non-blocking, so they scale better, though the "transaction" as a whole may take longer to fully settle.

**Use cases**: reach for 2PC in tightly coupled systems with a small number of participants, often within a single trusted infrastructure boundary — think a monolith talking to two databases, or short-lived financial ledger operations where you genuinely cannot tolerate the intermediate state being visible. Reach for Saga in microservice architectures spanning independently deployed, independently owned services, where availability and loose coupling matter more than strict atomicity, and where the business already has a natural notion of "cancel" or "refund."

### Real-World Example

The canonical example is exactly the one we opened with, translated into microservices: an e-commerce checkout flow with an Order Service, a Payment Service, and an Inventory Service, each owning its own database. This is almost always implemented as a saga in practice, not 2PC, because these services are independently deployed and you don't want a slow inventory check to hold a lock on payment infrastructure. Order is created in a "pending" state, payment is charged, inventory is reserved, and only then is the order marked "confirmed" — with cancellation, refund, and stock-release as the compensating actions if any step fails.

On the 2PC side, XA-style distributed transactions are still very real in enterprise settings — some relational databases and message brokers (for example, certain JMS-compliant brokers) support XA so that a single logical transaction can atomically write to a database and enqueue a message, guaranteeing you never lose a message or double-process one due to a partial failure. It's just far less common at internet scale because of the availability cost we discussed.

Companies like Netflix and Uber are frequently cited for saga-like patterns in their order and trip-booking flows — coordinating multiple services with compensations rather than distributed locks. And in practice, teams increasingly don't hand-roll saga orchestration logic themselves — they reach for workflow engines like **Temporal** or **Camunda**, which give you durable execution: you write orchestration code that looks like a normal function call sequence, and the engine persists state after every step so it can resume exactly where it left off after a crash, and it has first-class support for defining and automatically invoking compensating steps on failure.

### Recap

Let's bring it together. A distributed transaction problem arises whenever atomicity needs to span more than one independently-failing resource. Two-Phase Commit solves it by getting a unanimous vote from every participant before committing anything, giving you real atomicity and isolation, at the cost of blocking every participant if the coordinator dies mid-protocol — that's why it doesn't scale well and is rarely used across loosely coupled microservices. The Saga pattern solves it instead by committing each step locally and immediately, and using compensating transactions — a semantic undo, not a true rollback — to unwind partially-completed work if a later step fails. Sagas can be coordinated by choreography, using events with no central authority, or by orchestration, using a central coordinator that explicitly drives each step and each compensation. Sagas trade isolation and strict atomicity for availability and loose coupling, which is exactly the trade most modern microservice architectures are willing to make.

### What's Next

Now, sagas leave you with eventual consistency and a real question: if intermediate state is visible, and if a network hiccup makes a client retry a request, how do you make sure you don't double-charge a customer or double-reserve inventory? That's exactly what we're covering next — data consistency models and idempotency — so stick around, because it directly builds on everything we just covered about sagas.

## Key Takeaways

- ACID transactions rely on a single transaction manager with full local control; that assumption breaks the moment a transaction spans independent services or databases over an unreliable network.
- Two-Phase Commit uses a coordinator and participants: Phase 1 (Prepare/Vote) gets a durable commitment from everyone, Phase 2 (Commit/Abort) executes the unanimous decision.
- 2PC is a blocking protocol — if the coordinator crashes after participants vote yes but before the commit decision is delivered, participants are stuck holding locks indefinitely. This is a real availability cost, not a theoretical one.
- XA is the standard specification behind most real-world 2PC implementations, coordinating resources like relational databases and XA-compliant message brokers.
- The Saga pattern replaces one distributed atomic transaction with a sequence of local transactions plus compensating transactions that semantically undo prior steps on failure — compensations are new operations, not true rollbacks, and they don't restore isolation.
- Choreography-based sagas coordinate via events with no central authority (loosely coupled, harder to trace); orchestration-based sagas use a central orchestrator that explicitly drives steps and compensations (easier to reason about, but the orchestrator is critical infrastructure).
- 2PC favors strong consistency and isolation at the cost of availability; Saga favors availability and loose coupling at the cost of isolation and requiring careful, idempotent compensation logic.
- Tools like Temporal and Camunda provide durable execution engines purpose-built for implementing sagas in production without hand-rolling state persistence and retry logic.
