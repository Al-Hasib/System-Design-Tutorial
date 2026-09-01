# Study Notes: Batch vs Stream Processing

## Core Definitions

- **Batch processing**: processing a bounded, finite dataset collected over a period of time, all at once, as a discrete job with a start and end.
- **Stream processing**: processing an unbounded, continuous flow of data, event by event (or in small micro-batches), as it arrives.
- **Windowing**: the technique of dividing an infinite stream into finite chunks so aggregate computations (counts, averages, etc.) can be computed.
- **Event time**: the timestamp of when an event actually occurred.
- **Processing time**: the timestamp of when the system actually processes the event.
- **Watermark**: a mechanism in stream processing that estimates how far behind in event time the system may still receive data, used to decide when to finalize a window despite possible late arrivals.

## Batch vs Stream Comparison

| Aspect | Batch Processing | Stream Processing |
|---|---|---|
| Data scope | Bounded, finite dataset | Unbounded, continuous data |
| Latency | High (minutes to hours/days) | Low (milliseconds to seconds) |
| Throughput efficiency | Very high (bulk optimization) | Lower per-event efficiency, but scalable continuously |
| Complexity | Lower (simpler mental model) | Higher (ordering, late data, windowing) |
| Typical tools | Hadoop MapReduce, Apache Spark (batch mode), Apache Hive | Kafka Streams, Apache Flink, Spark Structured Streaming |
| Example use cases | Nightly financial reports, weekly ML model retraining, payroll processing | Fraud detection, real-time dashboards, alerting, live recommendations |
| Data freshness | Stale by definition (as old as the batch interval) | Near real-time |

## Windowing Types

| Window Type | Description | Example |
|---|---|---|
| Tumbling | Fixed-size, non-overlapping, back-to-back windows | "Orders per 60-second window" |
| Sliding | Fixed-size windows that overlap, recomputed at a smaller interval | "Last 60 seconds, updated every 10 seconds" |
| Session | Groups events by activity, closes after a gap of inactivity | "User session ends after 30 min idle" |

## Lambda vs Kappa Architecture

| Aspect | Lambda Architecture | Kappa Architecture |
|---|---|---|
| Approach | Parallel batch layer (accurate, slow) + speed layer (fast, approximate); merge results at serving time | Single stream-processing pipeline; treat all data (including historical) as a replayable stream |
| Codebases | Two separate codebases/logic paths to maintain | One codebase |
| Reprocessing | Batch layer periodically recomputes/corrects | Replay the stream from the beginning to recompute |
| Complexity | Higher operational burden (two systems) | Simpler, but requires a log-based system (e.g., Kafka) with sufficient retention |
| Enabled by | Traditional batch (Hadoop) + streaming systems side by side | Durable, replayable logs like Kafka |

## Quick Summary

- Choose batch when latency of hours/days is acceptable and you want maximum throughput/efficiency for large, bounded computations.
- Choose stream when you need to react within seconds and can tolerate the added complexity of ordering, late data, and windowing.
- Many real systems (e.g., Spotify, Uber) use both, matched to different use cases within the same product.
- Kappa architecture is increasingly favored over Lambda where a durable, replayable log (like Kafka) is already part of the stack, since it avoids maintaining two parallel codebases.
