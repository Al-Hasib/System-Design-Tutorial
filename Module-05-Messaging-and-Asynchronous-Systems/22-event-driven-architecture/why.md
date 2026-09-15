# Why This Topic Matters: Event-Driven Architecture

> **In one sentence:** Request-driven systems ask "who do I need to call?"; event-driven systems announce "this happened" — and that inversion is what lets a system keep growing without every change rippling through every service.

## The World Before This Idea

In a request-driven system, business processes are encoded as call chains. Checkout calls payment, which calls fraud, which calls the risk service, which calls the ledger. The chain is synchronous, so total latency is the sum of every hop and total availability is the product of every service's availability. Five services at 99.9% each gives you 99.5% — 3.6 hours of downtime a month, from arithmetic alone.

And the business process lives nowhere. There's no single place that describes "what happens when an order is placed" — it's distributed across a call graph you can only reconstruct by reading a dozen repositories.

## The Problems It Solves

### 1. Availability that degrades with every dependency
**What you see:** Your service is up 99.99% of the time and your users experience far less, because you're only as available as everything you synchronously call.

**Why it happens:** Synchronous chains multiply failure probabilities and add latencies.

**How EDA solves it:** Services publish and consume events asynchronously. A downstream service being down means its events pile up, not that the upstream operation fails. Availability stops being a product and starts being per-service.

### 2. Adding capability requires modifying existing code
**What you see:** Every new feature touches the core services, so the core services are always in conflict, always being reviewed by teams who don't own the feature.

**Why it happens:** Orchestration logic concentrates in callers.

**How EDA solves it:** New behavior is a new consumer of existing events. Adding fraud scoring, a loyalty program, or a data pipeline requires zero changes to the order service. Systems that grow by *addition* rather than *modification* stay tractable much longer.

### 3. No record of what actually happened
**What you see:** A customer disputes a charge and you can reconstruct only the current state, not the sequence of decisions that produced it. A bug corrupted data last Tuesday and you can't tell what the correct value should be.

**Why it happens:** State-mutating systems overwrite history. The database records what *is*, never what *happened*.

**How EDA (and event sourcing) solves it:** The event log is an immutable, ordered record of every fact. You can audit, debug by replay, rebuild a corrupted read model from scratch, and answer questions nobody thought to ask when the schema was designed. For regulated domains this is worth the entire architecture on its own.

### 4. New consumers that need historical data
**What you see:** You build a recommendation engine and want to train it on two years of behavior that was never stored in a usable form.

**Why it happens:** Request-driven systems don't retain the stream of what happened, only the result.

**How EDA solves it:** A durable log lets a brand-new consumer start at offset zero and process all of history to build its own view. That capability — bootstrapping a new service from the past — is genuinely difficult to get any other way.

## The Price You Pay

Event-driven architecture is a serious commitment, and it fails badly when adopted casually:

- **There is no global "now."** Different services have different views at any instant. Some of that is invisible to users; some of it is a correctness problem you must design around explicitly.
- **Control flow is invisible.** You cannot read a function and know what happens next. Understanding a business process means understanding subscriptions across the whole system — the single biggest complaint from teams who've adopted it.
- **Debugging requires infrastructure.** Without correlation IDs, distributed tracing, and centralized logs, diagnosing a failure that spans eight asynchronous hops is close to hopeless. Build the observability first, not after.
- **Event schemas are permanent public contracts.** Once published and consumed by unknown parties, an event's shape is very hard to change. Versioning discipline is required from day one.
- **Duplicates and reordering are guaranteed.** Every consumer must be idempotent and, usually, tolerant of out-of-order arrival.
- **It's the wrong default for small systems.** A five-person team building a CRUD product will move faster and sleep better with synchronous calls inside a monolith. EDA earns its cost at organizational scale, not at code scale.

## When You Need It — and When You Don't

| Go event-driven when | Stay request-driven when |
|---|---|
| Many independent teams must evolve in parallel | One small team owns everything |
| Reactions to an action keep multiplying | The workflow is short, fixed, and needs an immediate answer |
| Audit trails or replay are genuine requirements | Strong consistency is required across the whole operation |
| You need to buffer bursts and decouple availability | Debugging simplicity matters more than decoupling |
| Consumers have very different processing rates | You lack tracing and monitoring maturity |

## Why This Shows Up in Interviews

EDA appears in designs with many reacting subsystems: e-commerce checkout, ride-sharing state transitions, notification platforms, and analytics pipelines. Interviewers look for whether you can articulate what you *gain* (decoupling, independent scaling, auditability, resilience) and — more tellingly — what you *lose* (immediate consistency, traceability, simple debugging). A candidate who proposes EDA and then says "but I'd keep payment authorization synchronous, because the user needs a definitive yes or no before I confirm the order" is showing exactly the judgment being assessed.

## How It Connects

EDA is built on **Pub/Sub** (topic 21) over **message queues** (topic 20), typically a log-based one like Kafka for replay. It's the communication backbone that makes **microservices** genuinely independent (topics 30-31), depends on **idempotency** and **eventual consistency** (topic 29), and implements cross-service workflows through the **Saga pattern** (topic 28). Its events feed **stream processing** (topic 23), and it makes **observability** (topic 43) a hard prerequisite. **Domain-driven design** (topic 32) is where good event boundaries come from.

**Next:** [Batch vs Stream Processing](../23-batch-vs-stream-processing/why.md) — two ways to actually process the events you're now producing.
