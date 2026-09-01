# Publish-Subscribe Pattern

**Difficulty:** Intermediate
**Estimated video length:** 12-16 min
**Prerequisites:** [20 - Message Queues Explained: Kafka vs RabbitMQ](../20-message-queues-kafka-vs-rabbitmq/README.md)

## Learning Objectives

- Define the publish-subscribe (pub-sub) pattern and how it differs from a point-to-point queue
- Explain topics, publishers, subscribers, and fan-out
- Understand how pub-sub achieves loose coupling between producers and many independent consumers
- Recognize common pub-sub implementations and when to use them
- Identify the tradeoffs pub-sub introduces (message ordering, delivery guarantees, discoverability)

## Script

### Hook/Intro

Think about a YouTube channel — this one, actually. When I upload a video, I have no idea who's watching. I don't send you a personal message. I just publish the video, and everyone who's subscribed gets notified, whenever they check, on whatever device they use. I don't manage a list of your phone numbers. I don't know if you watch it now or next week. That relationship — one publisher, many independent subscribers, zero direct coupling between them — is exactly what the publish-subscribe pattern is in distributed systems. Let's dig into it.

### Point-to-Point vs Publish-Subscribe

In the last video, we talked about message queues, and most of the examples were what's called "point-to-point" — a message goes into a queue, and exactly one consumer (or one consumer within a group) picks it up and processes it. That's great for distributing work: process this job, ship this order, resize this image. Each message is handled once.

Publish-subscribe flips that model. Instead of one consumer handling a message, potentially many independent consumers each receive their own copy of the same message. The producer — now called a **publisher** — doesn't send a message to a specific queue destined for a specific worker. It publishes to a **topic** (sometimes called a channel), and any number of **subscribers** who have registered interest in that topic get a copy, independently, without knowing about each other.

The key mental model shift: with a queue, publishing a message says "someone, do this work." With pub-sub, publishing a message says "this thing happened — anyone who cares, react."

### Anatomy of Pub-Sub: Fan-Out

The magic word here is **fan-out**. One event goes in, and it gets duplicated out to every interested subscriber. If three services are subscribed to a "user-signed-up" topic, all three receive that exact event the moment it's published — no coordination required between them, no code changes needed in the publisher to add a fourth subscriber later.

This is what makes pub-sub the backbone of loosely coupled systems. The publisher's only responsibility is to say "this happened." It has zero knowledge of who's listening or what they do with it. You could add a brand-new subscriber next month — say, a new team building a customer-loyalty feature that wants to know every time a user signs up — and the publisher's code doesn't change one bit. That's an enormous win for team autonomy in large organizations: teams can build and deploy independently, subscribing to events they care about without ever touching the producing service's codebase.

### Common Implementations

You'll see pub-sub show up in several forms. **Kafka**, which we covered last time, naturally supports pub-sub because multiple consumer groups can independently read the same topic — that's fan-out built into its log model. **Redis Pub/Sub** offers a lightweight, in-memory, fire-and-forget version — great for real-time notifications where you don't need durability, since messages aren't persisted; if a subscriber isn't connected at publish time, it simply misses the message. **Google Cloud Pub/Sub**, **AWS SNS**, and **Azure Service Bus Topics** are managed cloud services purpose-built for pub-sub at scale, often pairing a topic with per-subscriber queues so each subscriber gets durable, at-least-once delivery even if the subscriber goes offline temporarily. **MQTT**, common in IoT, is a lightweight pub-sub protocol designed for low-bandwidth, high-latency networks like sensor devices.

Notice the pattern: pub-sub is a conceptual model, not a single technology. What matters is understanding when you need "notify N interested parties" versus "distribute work to one of M workers."

### Tradeoffs to Know

Pub-sub isn't free. First, **ordering** gets trickier — if you have multiple publishers or a topic gets fanned across partitions, guaranteeing a global order across everything can be difficult or even undefined, especially in fire-and-forget systems. Second, **delivery guarantees vary wildly** by implementation — Redis Pub/Sub is at-most-once and drops messages if no one's listening, while Kafka and cloud pub-sub services offer durable, at-least-once delivery via persistence. Third, **discoverability and debugging get harder**: because the publisher doesn't know who's subscribed, tracing "what happens when this event fires" across a large system can require good documentation, event schemas, and observability tooling — otherwise you end up with a sprawling, implicit dependency graph that's hard to reason about. This is sometimes only half-jokingly called "the distributed monolith" if teams aren't disciplined about event contracts.

### Real-World Example

Consider a ride-sharing app when a driver's location updates. That single "location updated" event might need to reach: the rider's live map view, an ETA-recalculation service, a surge-pricing engine, and a fraud-detection system watching for GPS spoofing. With a plain point-to-point queue, you'd need the location service to know about all four consumers and send four separate messages, or worse, make four separate API calls. With pub-sub, the location service publishes one "location updated" event to a topic. All four services subscribe independently. Next quarter, if the company adds a fifth feature — say, a "share my live trip" link for friends and family — that team just subscribes to the same topic. Zero changes needed anywhere else.

### Recap

Let's bring it together. Publish-subscribe is a messaging pattern where one publisher broadcasts events to a topic, and any number of independent subscribers each receive their own copy — this is called fan-out. It's fundamentally different from point-to-point queues, where one message is handled by one consumer. Pub-sub is what enables true loose coupling and independent team scaling in large systems, implemented via tools like Kafka, Redis Pub/Sub, SNS, and Google Cloud Pub/Sub — each with different durability and ordering guarantees. The tradeoff is you lose some visibility into "who's listening," so strong event contracts and observability become essential.

### What's Next

Publish-subscribe is a building block. Next video, we're going to zoom out and talk about the bigger architectural philosophy built on top of it: event-driven architecture, where entire systems are designed around producing, reacting to, and chaining events rather than direct service-to-service calls. See you there.

## Key Takeaways

- Pub-sub broadcasts one event to many independent subscribers (fan-out), unlike point-to-point queues where one message goes to one consumer.
- Publishers have zero knowledge of subscribers — this loose coupling enables independent team development and easy addition of new consumers.
- Implementations range from lightweight/fire-and-forget (Redis Pub/Sub, MQTT) to durable, at-least-once (Kafka, AWS SNS, Google Cloud Pub/Sub).
- Ordering and delivery guarantees vary significantly by implementation — check the fine print before relying on either.
- Strong event schemas/contracts and observability are essential to avoid an untraceable web of implicit dependencies.
