# Event-Driven Architecture

**Difficulty:** Intermediate/Advanced
**Estimated video length:** 15-19 min
**Prerequisites:** [21 - Publish-Subscribe Pattern](../21-publish-subscribe-pattern/README.md), [20 - Message Queues Explained: Kafka vs RabbitMQ](../20-message-queues-kafka-vs-rabbitmq/README.md)

## Learning Objectives

- Define event-driven architecture (EDA) and distinguish it from request-driven architecture
- Understand key EDA building blocks: events, event producers, event consumers, event brokers/buses
- Compare event notification, event-carried state transfer, and event sourcing styles
- Recognize the benefits (scalability, decoupling, resilience) and challenges (complexity, eventual consistency, debugging) of EDA
- Apply EDA thinking to a realistic multi-service system design scenario

## Script

### Hook/Intro

Picture a newsroom versus a phone tree. In a phone tree, if you want ten people to know something, you call the first person, who calls the next, who calls the next — and if any one link is unavailable, the chain breaks. In a newsroom, a reporter just publishes the story. Every subscriber — every reader with a subscription — finds out on their own time, through their own channel, without the reporter needing to personally notify each one. Event-driven architecture is the newsroom model applied to software systems: instead of services calling each other directly and waiting for responses, they publish facts about things that happened, called events, and let other services react independently. Today we're zooming out from the individual patterns we've covered and looking at the whole architectural philosophy.

### Request-Driven vs Event-Driven

Most of us start out building request-driven systems: Service A calls Service B's API, waits for a response, and continues based on what B returns. This is simple to reason about — it's synchronous, linear, easy to trace. But it creates tight temporal coupling: A now depends on B being available, fast, and correct, at the exact moment A needs it.

Event-driven architecture inverts this. Instead of Service A asking Service B to do something and waiting, Service A announces "X happened" — an event — and doesn't know or care who reacts to it, or when. This isn't just the pub-sub mechanism we discussed last time; it's a whole way of designing your system's flow of control. In an event-driven system, the sequence of what happens next isn't hard-coded into any one service — it emerges from which services are listening for which events. That's a profound shift: your system's behavior becomes a graph of reactions to facts, rather than a chain of commands.

### Core Building Blocks

An event-driven system has a few key parts. An **event** is an immutable record that something happened — "OrderPlaced," "PaymentProcessed," "UserDeactivated." Note the past tense; events describe facts, not requests. Unlike a command, which says "do this," an event says "this occurred," and it's the listener's choice whether and how to react.

**Event producers** emit these events when something noteworthy happens in their domain. **Event consumers** subscribe to relevant events and react — updating their own data, triggering side effects, or emitting new events of their own, creating chains of reactions. And tying it together is an **event broker** or **event bus** — often Kafka, RabbitMQ, or a cloud-native service — that handles the actual delivery, buffering, and fan-out we covered in the last two videos.

### Three Flavors of Events

It's worth knowing there isn't just one style of "event" in EDA. **Event notification** is the lightest — the event just says something happened and maybe includes an ID, and if the consumer needs more detail, it calls back to the source service's API. This keeps events small but reintroduces some coupling, since consumers still depend on the producer's API being available later.

**Event-carried state transfer** goes further — the event itself carries all the relevant data the consumer needs, so it never has to call back. For example, an "OrderPlaced" event might include the full order details, customer ID, and item list, so the shipping service doesn't need to query the order service at all. This maximizes decoupling and resilience but means events carry more data and you need to think carefully about keeping that data reasonably fresh and consistent.

**Event sourcing** is the most architecturally ambitious: instead of storing just the current state of an entity, you store the entire sequence of events that led to that state, and the current state is derived by replaying them. Think of it like a bank ledger instead of a bank balance — you don't overwrite the balance, you record every transaction, and the balance is a computed view. This gives you a full audit trail and the ability to rebuild state at any point in time, at the cost of significant complexity in querying and tooling.

### Benefits and Challenges

The benefits of EDA echo what we've built up over the last two videos: services scale independently, a failure in one consumer doesn't take down the producer or other consumers, and new functionality can be added by simply subscribing to existing events — no changes to upstream services required. This is a huge deal for large organizations with many autonomous teams.

But EDA isn't free lunch. The biggest challenge is **eventual consistency**: because reactions happen asynchronously, there's a window of time where different parts of the system have different views of reality. If a customer support agent looks at an order two hundred milliseconds after it was placed, has inventory been decremented yet? Has the confirmation email gone out? You have to design your system — and your product expectations — around this lag, rather than pretending it doesn't exist.

The second challenge is **complexity and debuggability**. When behavior emerges from a web of event producers and consumers rather than a single call stack, tracing "why did this happen" across dozens of services requires serious investment in distributed tracing, correlation IDs, and event schemas. Teams that adopt EDA without this investment often end up with systems that are technically decoupled but practically impossible to debug.

### Real-World Example

Think about how a company like Netflix handles a user finishing an episode. That single fact — "playback completed" — triggers a cascade of independent reactions: the recommendation engine updates viewing history to refine what to suggest next, the "continue watching" row gets updated, a billing/usage analytics service logs watch time for licensing calculations, and a autoplay-next-episode feature kicks in. None of these are chained together in a single service's code. The playback service just emits one event, and this constellation of teams, each with different priorities and release schedules, reacts on their own terms. If the recommendation engine is being redeployed and briefly unavailable, playback isn't affected at all — it'll simply catch up on the event once it's back.

### Recap

Let's recap. Event-driven architecture is a design philosophy where system behavior emerges from services producing and reacting to events, rather than services directly calling and waiting on each other. Events are immutable facts about things that already happened. There are three flavors: lightweight event notification, richer event-carried state transfer, and the most ambitious, event sourcing, where the event log itself is the source of truth. EDA buys you scalability, resilience, and team autonomy, but costs you eventual consistency and added system complexity that demands strong observability practices.

### What's Next

We've now covered the "how" of moving events and the "why" of designing around them. Next video, we're going to look at a specific and very practical decision that shows up constantly in event-driven and data-heavy systems: do you process your data in batches, on a schedule, or continuously as a stream, the instant it arrives? We'll compare batch and stream processing head to head. See you there.

## Key Takeaways

- Event-driven architecture (EDA) inverts request-driven design: services announce facts (events) instead of directly calling and waiting on each other.
- Events are immutable, past-tense records of something that happened, produced and consumed independently through an event broker/bus.
- Three event styles: event notification (thin, may require callback), event-carried state transfer (self-contained), and event sourcing (event log as source of truth).
- EDA benefits: independent scaling, resilience to individual service failure, and easy addition of new functionality without touching upstream services.
- EDA costs: eventual consistency (temporary inconsistent views across services) and higher system complexity requiring strong observability and event contracts.
