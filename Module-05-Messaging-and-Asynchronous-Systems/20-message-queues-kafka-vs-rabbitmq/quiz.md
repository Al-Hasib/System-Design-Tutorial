# Practice & Interview Questions

**1. Why would you introduce a message queue between two services instead of having them call each other directly (e.g., via REST)?**
A message queue decouples the services in time, space, and speed. The producer doesn't need the consumer to be online or fast, so a slow or temporarily unavailable downstream service doesn't cause the producer's request to fail or hang. It also absorbs traffic spikes and allows each service to scale independently.

**2. What is the fundamental architectural difference between Kafka and RabbitMQ?**
RabbitMQ is a traditional message broker where messages are routed through exchanges into queues and deleted once consumed and acknowledged. Kafka is a distributed, append-only log where messages are retained for a configurable period regardless of consumption, and multiple independent consumer groups can each read (and replay) the same data at their own offsets.

**3. When would you choose RabbitMQ over Kafka?**
When you need complex routing logic (e.g., route by message type or priority), classic task/job-queue semantics where each job should be picked up by exactly one worker, lower operational overhead, or per-message acknowledgment and retry semantics rather than high-throughput event streaming.

**4. When would you choose Kafka over RabbitMQ?**
When you need very high throughput event ingestion, durable replay of historical events, multiple independent teams/services consuming the same event stream, or you're building an event-sourcing/stream-processing pipeline (e.g., feeding both real-time dashboards and a data lake from the same events).

**5. How do you guarantee at-least-once vs exactly-once delivery?**
At-least-once is achieved by having the broker redeliver a message until it receives an acknowledgment — this risks duplicates if the ack is lost after successful processing. Exactly-once requires additional machinery: idempotent producers (deduplicating retries), transactional writes that atomically commit both the message and the consumer's offset, or, more practically, at-least-once delivery combined with an idempotent consumer (e.g., checking a unique message ID before applying an update) so re-processing has no side effect.

**6. What is a consumer group and why does it matter for scaling?**
A consumer group is a set of consumer instances that cooperatively consume a queue or topic, splitting the work so each message/partition is handled by only one instance in the group at a time. It lets you scale processing horizontally — add more consumer instances (up to the number of partitions in Kafka) to increase throughput.

**7. What happens if a consumer crashes after reading a message but before acknowledging it?**
The broker doesn't receive the acknowledgment, so it considers the message unprocessed and redelivers it — either to another consumer in the group or back to the same one after it recovers. This is the mechanism behind at-least-once delivery, and it's why consumers must be designed to safely handle duplicate deliveries.

**8. Explain backpressure in the context of message queues.**
Backpressure is what occurs when producers generate messages faster than consumers can process them. A queue absorbs this mismatch by buffering messages instead of overwhelming the consumer or causing requests to fail, letting the consumer catch up at its own sustainable pace.

**9. Why can Kafka support "replaying" old events while a typical RabbitMQ queue cannot?**
Kafka stores messages as an immutable, ordered log on disk for a configured retention window (time or size based) and consumers track their own read offset rather than the broker deleting on read. RabbitMQ queues are consumption-based — once a message is acknowledged, it is removed, so there's nothing left to replay unless you explicitly persist it elsewhere.

**10. Scenario: You're building a system where an "order placed" event needs to trigger inventory updates, send a confirmation email, and update a fraud-detection model, and a new analytics team may want to add their own consumer next quarter without touching existing code. Which technology fits better and why?**
Kafka fits better here because you have multiple independent consumers reading the same event, and you want to add new consumers later without modifying the producer or existing consumers. Kafka's topic/consumer-group model lets each team subscribe independently and even replay historical events when they onboard.

**11. What is message ordering, and how do Kafka and RabbitMQ each handle it?**
Ordering means messages are processed in the sequence they were produced. Kafka guarantees ordering only within a single partition (messages with the same key go to the same partition and are read in order); across partitions there's no global order. RabbitMQ guarantees FIFO ordering within a single queue, assuming a single consumer; with multiple competing consumers, per-message ordering across the whole queue isn't guaranteed either.

**12. What operational tradeoff should you consider before choosing Kafka for a small-scale project?**
Kafka has higher operational complexity — it typically requires managing a cluster (brokers, partitions, replication, and either ZooKeeper or KRaft consensus), which can be overkill for low-throughput use cases. A simpler broker like RabbitMQ (or even a managed cloud queue) may be far cheaper to operate for smaller workloads.
