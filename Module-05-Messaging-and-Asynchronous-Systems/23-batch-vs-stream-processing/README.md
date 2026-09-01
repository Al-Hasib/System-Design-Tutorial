# Batch Processing vs Stream Processing

**Difficulty:** Intermediate/Advanced
**Estimated video length:** 14-18 min
**Prerequisites:** [22 - Event-Driven Architecture](../22-event-driven-architecture/README.md), [20 - Message Queues Explained: Kafka vs RabbitMQ](../20-message-queues-kafka-vs-rabbitmq/README.md)

## Learning Objectives

- Define batch processing and stream processing and identify their core differences
- Understand latency, throughput, and complexity tradeoffs between the two models
- Recognize windowing concepts in stream processing (tumbling, sliding, session windows)
- Identify common tools for each approach (Hadoop/Spark for batch, Kafka Streams/Flink/Spark Streaming for stream)
- Apply the Lambda and Kappa architecture concepts to real system design decisions

## Script

### Hook/Intro

Picture two ways of doing laundry. Option one: you wait until you've got a full basket, then run one big load — efficient, uses the machine at full capacity, but you might wait a week to have a clean shirt ready. Option two: you wash each item the moment it gets dirty, one at a time — you always have a clean shirt within minutes, but you're running the machine constantly and it's a lot less efficient per item. That's the entire tension between batch processing and stream processing, and today we're breaking down when each one is the right call.

### What Is Batch Processing?

Batch processing means collecting data over a period of time — an hour, a day, a week — and then processing it all together in one big job. Think of a nightly job that takes all of yesterday's transactions and generates a financial report, or a weekly job that recalculates a machine learning model using the last month of user behavior data. The defining trait is that batch processing works on a bounded, finite dataset — it has a clear beginning and end, and the job runs, finishes, and produces a result.

Batch processing is fantastic for throughput. Because you're processing everything at once, you can optimize heavily — sort data, use efficient bulk algorithms, parallelize across many machines, and squeeze maximum efficiency per unit of data processed. The classic example is Hadoop's MapReduce, or more modern tools like Apache Spark running batch jobs. The tradeoff is latency: the result isn't available until the whole batch finishes. If your batch runs once a night, your data is, by definition, always at least a few hours stale.

### What Is Stream Processing?

Stream processing flips this: instead of waiting to accumulate a batch, you process each piece of data — each event — as it arrives, continuously, with no defined "end" to the dataset. Data is treated as an unbounded, ever-flowing stream, and your processing logic runs on each new event (or micro-batch of events) within milliseconds to seconds of it happening.

This is what powers things like real-time fraud detection — you can't wait until tonight's batch job to flag a stolen credit card, you need to catch it in the two seconds between swipe and approval. Or live dashboards showing "users online right now." Or triggering an alert the instant a server's error rate spikes. Tools here include Kafka Streams, Apache Flink, and Spark Structured Streaming.

The tradeoff is complexity and, often, lower raw throughput efficiency per event compared to batch, because you're not getting the benefit of processing things in bulk. Stream processing systems also have to grapple with a genuinely hard problem: events don't always arrive in order, and they don't always arrive on time.

### Windowing: Making Sense of an Infinite Stream

Since a stream never "ends," how do you compute something like "average orders per minute"? You need to slice the infinite stream into finite chunks, and this is called **windowing**. A **tumbling window** breaks the stream into fixed, non-overlapping chunks — say, every 60 seconds, calculate the count, then start fresh. A **sliding window** overlaps — for example, "the last 60 seconds," recalculated every 10 seconds, so windows share data. A **session window** groups events based on activity gaps — for example, grouping a user's clicks into one "session" as long as they're active, and closing the window after, say, 30 minutes of inactivity. Choosing the right windowing strategy is central to designing correct real-time analytics.

There's also the thorny issue of **event time vs processing time**. Event time is when something actually happened; processing time is when your system got around to handling it. Network delays, retries, and out-of-order delivery mean these can diverge, and sophisticated stream processors use concepts like watermarks to decide "how long do we wait for late data before we consider a window closed and finalize its result."

### Lambda and Kappa Architecture

Historically, many companies wanted both: the accuracy and reprocessability of batch, and the low latency of streaming. This led to the **Lambda architecture**: run the same data through two parallel paths — a batch layer that periodically computes accurate, comprehensive results, and a speed layer that computes fast, approximate real-time results — then merge the two views when serving queries, letting the batch layer eventually correct any drift from the speed layer. It works, but you're maintaining two separate codebases doing similar logic, which is a real operational burden.

The **Kappa architecture** was proposed as a simplification: treat everything as a stream, including historical data, and just reprocess the stream from the beginning when you need to recompute something, rather than maintaining a wholly separate batch pipeline. This is only practical because systems like Kafka can retain and replay historical data as a log — which, hopefully, sounds familiar from our very first video in this module. Kappa trades some of Lambda's flexibility for a dramatically simpler single-codebase system.

### Real-World Example

Think about a company like Spotify. They need both models simultaneously. Batch processing generates your "Discover Weekly" playlist — it's fine that this runs once a week, crunching massive historical listening data with heavy, expensive machine learning models, because freshness of "once a week" is perfectly acceptable for that product. But stream processing is what powers the "currently playing" count you might see on an artist's page, or real-time detection of a song going viral so it can be promoted immediately — that can't wait for a nightly batch job; it needs to react within seconds.

### Recap

Let's wrap up. Batch processing works on bounded, finite datasets, optimizing for throughput and efficiency at the cost of latency — perfect for periodic, large-scale computation where staleness of hours or days is acceptable. Stream processing works on unbounded, continuous data, processing each event with low latency, and requires dealing with windowing and out-of-order/late data — perfect when you need to react to something within seconds. Lambda architecture runs both in parallel to get the best of each; Kappa architecture simplifies by treating everything as a replayable stream. Choosing between them isn't about which is "better" — it's about matching the freshness requirement of your use case to the right tool.

### What's Next

That wraps up Module 5 on messaging and asynchronous systems — we've gone from the basic mechanics of queues, to fan-out with pub-sub, to the big-picture philosophy of event-driven architecture, and now to how you actually process the data flowing through all of it. In the next module, we shift gears to distributed systems concepts — starting with consistent hashing, one of the most elegant ideas in all of distributed systems design. See you there.

## Key Takeaways

- Batch processing handles bounded, finite datasets on a schedule, optimizing for throughput at the cost of latency (data freshness measured in hours/days).
- Stream processing handles unbounded, continuous data as it arrives, optimizing for low latency (seconds or less) at some cost to raw processing efficiency.
- Windowing (tumbling, sliding, session) is how stream processing slices an infinite stream into computable chunks.
- Event time vs processing time and out-of-order/late data are core challenges unique to stream processing.
- Lambda architecture runs parallel batch + speed layers for accuracy and low latency; Kappa architecture simplifies to a single replayable stream pipeline.
- The right choice depends on how fresh your results need to be, not which approach is inherently "better."
