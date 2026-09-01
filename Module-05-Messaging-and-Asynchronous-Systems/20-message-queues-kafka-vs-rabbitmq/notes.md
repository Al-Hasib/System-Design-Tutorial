# Study Notes: Message Queues, Kafka vs RabbitMQ

## Core Definitions

- **Message queue**: an intermediary that stores messages sent by producers until consumers are ready to process them, enabling asynchronous communication.
- **Producer**: a service/client that creates and sends messages.
- **Broker**: the server(s) that receive, store, and route messages.
- **Consumer**: a service/client that reads and processes messages.
- **Acknowledgment (ack)**: a signal from consumer to broker confirming a message was processed, allowing safe removal or offset advancement.
- **Backpressure**: the queue absorbing bursts of production faster than consumption, protecting downstream systems from overload.
- **Consumer group**: a set of consumer instances that share the work of consuming a queue/topic, enabling horizontal scaling.
- **Idempotency**: designing consumer logic so processing the same message twice produces the same result as processing it once.

## Why Asynchronous Messaging?

- Decouples in **time**: consumer doesn't need to be online when message is sent.
- Decouples in **space**: producer doesn't need to know who/how many consumers exist.
- Decouples in **speed**: a slow consumer doesn't block or slow the producer.
- Prevents cascading failures: one dependency being down doesn't take down the whole request chain.
- Enables independent scaling of producers and consumers.

## Kafka vs RabbitMQ

| Aspect | RabbitMQ | Kafka |
|---|---|---|
| Model | Traditional message broker (smart broker, dumb consumer) | Distributed append-only log (dumb broker, smart consumer) |
| Message lifecycle | Deleted after consumption + ack | Retained for a configured period regardless of consumption |
| Replay | Not natively supported | Native — consumers can rewind/replay by resetting offsets |
| Routing | Rich routing via exchanges (direct, topic, fanout, headers) | Simpler — topic + partition based |
| Throughput | High, but generally lower than Kafka at extreme scale | Very high throughput, built for millions of events/sec |
| Ordering | Per-queue ordering | Per-partition ordering guarantee |
| Multiple independent consumers of same data | Harder (message removed once consumed) | Natural — each consumer group tracks its own offset |
| Best fit | Task queues, RPC-style workflows, complex routing | Event streaming, event sourcing, log aggregation, analytics pipelines |
| Protocol | AMQP (also supports MQTT, STOMP) | Custom binary protocol over TCP |
| Operational complexity | Lower to moderate | Higher (needs ZooKeeper/KRaft, partition management) |

## Delivery Guarantees

| Guarantee | Description | Risk |
|---|---|---|
| At-most-once | Message sent once, no retry | Message loss possible |
| At-least-once | Broker retries until acked | Duplicate processing possible |
| Exactly-once | Each message processed exactly once | Hard to achieve across system boundaries; Kafka supports it within its own pipeline via idempotent producers + transactions |

- Practical default in most real systems: **at-least-once delivery + idempotent consumers**.

## Scaling Mechanisms

- **Partitioning** (Kafka): a topic is split into partitions distributed across brokers; each partition is an ordered, append-only log.
- **Consumer groups**: within a group, each partition is consumed by exactly one consumer instance at a time — this is how Kafka parallelizes consumption.
- **Prefetch / QoS** (RabbitMQ): controls how many unacknowledged messages a consumer can hold, balancing throughput and fairness.

## Quick Summary

- Use a queue when you need reliable, decoupled, asynchronous communication between services.
- Choose RabbitMQ for complex routing and classic task/job queues.
- Choose Kafka for high-throughput event streaming, replay, and multiple independent consumer applications reading the same data.
- Always design consumers to be idempotent — assume at-least-once delivery in production systems.
