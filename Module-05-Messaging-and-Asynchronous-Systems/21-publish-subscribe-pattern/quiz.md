# Practice & Interview Questions

**1. What is the core difference between a point-to-point queue and publish-subscribe?**
In a point-to-point queue, each message is consumed by exactly one consumer (or one within a consumer group). In publish-subscribe, each message is delivered independently to every subscriber of a topic — this duplication is called fan-out.

**2. Why does pub-sub reduce coupling compared to direct service-to-service calls?**
The publisher only needs to know about the topic it publishes to, not the identity, number, or behavior of any subscribers. Subscribers can be added or removed without any change to the publisher's code, which decouples the systems and allows independent development and deployment.

**3. Give an example of when you'd choose Redis Pub/Sub over Kafka for a pub-sub use case.**
Redis Pub/Sub is a good fit for lightweight, real-time, low-stakes notifications (e.g., live cursor positions in a collaborative editor, or transient chat "typing" indicators) where losing a message occasionally is acceptable and you don't need durability or replay. Kafka is overkill here due to its operational complexity and persistence overhead.

**4. What happens to a message in Redis Pub/Sub if no subscriber is currently connected?**
It is lost. Redis Pub/Sub is fire-and-forget and at-most-once — it does not persist messages, so any subscriber that isn't actively connected at publish time simply never receives that message.

**5. How do AWS SNS and Google Cloud Pub/Sub typically achieve durable, at-least-once delivery unlike Redis Pub/Sub?**
They persist published messages and, in many setups, pair each subscriber with its own durable queue (e.g., SNS commonly fans out to per-subscriber SQS queues) so that even if a subscriber is offline temporarily, the message waits until it can be delivered and acknowledged, retrying as needed.

**6. Why can global message ordering be difficult to guarantee in a pub-sub system?**
When there are multiple publishers, multiple partitions, or multiple subscribers processing in parallel, there's no single, universally agreed sequence of events across the whole system — each subscriber may see messages in a different relative timing, and some implementations don't guarantee any ordering at all across a fanned-out topic.

**7. What operational/organizational risk does pub-sub introduce, and how do teams mitigate it?**
Because publishers don't know who consumes their events, it becomes hard to trace what happens when an event fires, leading to an implicit and potentially sprawling dependency graph (sometimes called a "distributed monolith"). Teams mitigate this with well-documented event schemas/contracts, event catalogs, and distributed tracing/observability tooling.

**8. Scenario: A social media platform needs to notify a user's followers whenever they post. New features (push notifications, activity feed generation, trending-topic detection) are added by different teams over time. How would you design this with pub-sub, and why not point-to-point?**
Publish a "post-created" event to a topic when a user posts. Each feature team (push notifications, feed generation, trending detection) subscribes independently. A point-to-point queue would force the posting service to know about and directly message each downstream feature, requiring code changes in the producer every time a new feature/team is added — pub-sub avoids that coupling entirely.

**9. What is the relationship between Kafka consumer groups and the pub-sub pattern?**
Each consumer group in Kafka acts like an independent subscriber to a topic — within a group, work is load-balanced across partitions (point-to-point style), but across different groups, the same message is delivered to each group independently, which is exactly the fan-out behavior of pub-sub. Kafka effectively supports both patterns depending on how consumer groups are configured.

**10. What is MQTT and why is it commonly used with pub-sub in IoT contexts?**
MQTT is a lightweight publish-subscribe messaging protocol designed for constrained devices and low-bandwidth, high-latency, or unreliable networks. Its small message overhead and configurable Quality of Service (QoS) levels make it well suited for scenarios like sensors publishing telemetry data to a broker that fans it out to multiple monitoring or automation subscribers.

**11. Why might a fire-and-forget pub-sub system be inappropriate for a financial transaction event, and what would you use instead?**
Financial transaction events typically must not be lost — a dropped "payment completed" event could mean a customer is charged but downstream systems never record it. You'd use a durable, at-least-once pub-sub implementation (e.g., Kafka or SNS/SQS) combined with idempotent consumers, rather than a fire-and-forget system like plain Redis Pub/Sub.

**12. How would you add a brand-new consumer to an existing pub-sub topic without any risk to existing subscribers?**
Simply have the new service subscribe to the existing topic (or, in Kafka's case, create a new consumer group reading from the beginning or current offset of the topic). Because subscribers are independent, this requires no changes to the publisher or to any other subscriber, and it carries no risk of disrupting their processing.
