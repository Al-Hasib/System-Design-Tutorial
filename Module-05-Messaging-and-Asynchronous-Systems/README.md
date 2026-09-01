# Module 5: Messaging & Asynchronous Systems

This module covers how large-scale systems communicate without talking to each other directly: message queues, publish-subscribe fan-out, event-driven architectures, and the tradeoffs between batch and stream data processing. These patterns let services decouple in time and space — a producer doesn't need the consumer to be online, fast, or even aware it exists — which is what makes systems resilient to traffic spikes, partial failures, and independent team deployments. Mastering this module is essential for designing systems that scale horizontally without becoming a tangle of brittle synchronous calls.

## Videos in This Module

| # | Title | Description | Link |
|---|-------|-------------|------|
| 20 | Message Queues Explained: Kafka vs RabbitMQ | How message queues decouple producers and consumers, and when to reach for Kafka vs RabbitMQ | [20-message-queues-kafka-vs-rabbitmq](./20-message-queues-kafka-vs-rabbitmq/README.md) |
| 21 | Publish-Subscribe Pattern | How pub-sub fans out one event to many independent subscribers | [21-publish-subscribe-pattern](./21-publish-subscribe-pattern/README.md) |
| 22 | Event-Driven Architecture | Designing systems around events instead of direct calls | [22-event-driven-architecture](./22-event-driven-architecture/README.md) |
| 23 | Batch Processing vs Stream Processing | Choosing between processing data in batches or as a continuous stream | [23-batch-vs-stream-processing](./23-batch-vs-stream-processing/README.md) |
