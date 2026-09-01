# Study Notes: Publish-Subscribe Pattern

## Core Definitions

- **Publish-Subscribe (pub-sub)**: a messaging pattern where publishers broadcast messages to a topic/channel, and any number of independent subscribers receive their own copy of each message.
- **Publisher**: sends messages to a topic without knowledge of who (if anyone) is subscribed.
- **Subscriber**: registers interest in a topic and receives a copy of every message published to it.
- **Topic / Channel**: the named destination publishers write to and subscribers read from.
- **Fan-out**: the process of delivering one published message to multiple subscribers simultaneously.

## Queue (Point-to-Point) vs Pub-Sub

| Aspect | Point-to-Point Queue | Publish-Subscribe |
|---|---|---|
| Delivery | One message consumed by exactly one consumer (or one per consumer group) | One message delivered to every subscriber independently |
| Use case | Distributing discrete work/jobs among workers | Broadcasting an event/notification to many interested parties |
| Coupling | Producer may care that work gets done once | Producer has zero knowledge of subscriber count or identity |
| Scaling model | Add more consumers to share the load | Add more subscribers to react to the same event independently |
| Example | Image resizing job queue | "user signed up" event notifying email, analytics, loyalty services |

## Common Pub-Sub Implementations

| Technology | Durability | Delivery Guarantee | Typical Use |
|---|---|---|---|
| Redis Pub/Sub | None (in-memory, fire-and-forget) | At-most-once; missed if subscriber offline | Real-time, low-stakes notifications |
| Apache Kafka | Durable log, replayable | At-least-once (configurable, exactly-once within Kafka) | High-throughput event streaming, multiple independent consumer groups |
| AWS SNS | Durable, often paired with SQS per subscriber | At-least-once | Fan-out to multiple AWS services/queues |
| Google Cloud Pub/Sub | Durable | At-least-once | Managed, global-scale event distribution |
| Azure Service Bus Topics | Durable | At-least-once | Enterprise messaging with subscription filters |
| MQTT | Varies by QoS level (0/1/2) | At-most-once to exactly-once depending on QoS | IoT, low-bandwidth sensor networks |

## Why Pub-Sub Matters at Scale

- Enables **loose coupling**: publisher doesn't need to change when new subscribers are added.
- Enables **team autonomy**: different teams can subscribe to the same event stream independently.
- Supports **one-to-many** communication patterns natively, which point-to-point queues don't.

## Tradeoffs / Risks

- **Ordering** can be difficult to guarantee globally, especially with multiple publishers or partitioned topics.
- **Delivery guarantees vary** — some implementations drop messages if no subscriber is listening (fire-and-forget); others persist and guarantee at-least-once.
- **Discoverability/debugging** is harder: publisher doesn't know who consumes its events, so tracing effects across a system requires good documentation, schemas, and observability.
- Risk of an implicit, hard-to-trace dependency graph ("distributed monolith") without strong event contracts.

## Quick Summary

- Use pub-sub when one event needs to reach multiple independent consumers that shouldn't be coupled to each other or to the publisher.
- Use a point-to-point queue when you need exactly one worker to handle each unit of work.
- Choose the underlying technology based on durability and delivery-guarantee needs, not just "pub-sub vs queue" terminology.
