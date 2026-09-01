# Practice & Interview Questions

**1. What is the fundamental difference between batch processing and stream processing?**
Batch processing operates on a bounded, finite dataset collected over a period of time and processes it all at once as a discrete job. Stream processing operates on an unbounded, continuous flow of data, processing each event (or micro-batch) as it arrives, with no defined end to the dataset.

**2. Why does batch processing generally achieve higher throughput efficiency than stream processing?**
Because batch processing works on the entire dataset at once, it can apply bulk optimizations — sorting, efficient parallel algorithms, and full utilization of compute resources across a large, known amount of data — rather than paying per-event overhead for handling each item individually and continuously as in streaming.

**3. What is windowing in stream processing, and why is it necessary?**
Windowing is the technique of dividing an unbounded stream into finite chunks (windows) so that aggregate computations like counts, sums, or averages can be calculated. It's necessary because a stream has no natural end, so without windowing there's no way to define what "an average" or "a count" is being computed over.

**4. Explain the difference between a tumbling window and a sliding window.**
A tumbling window divides the stream into fixed-size, non-overlapping, back-to-back intervals (e.g., every distinct 60-second block). A sliding window is also fixed-size but overlaps with adjacent windows, recalculated at a smaller interval than its size (e.g., a 60-second window recomputed every 10 seconds), so the same events can contribute to multiple windows.

**5. What is the difference between event time and processing time, and why does it matter?**
Event time is when something actually happened in the real world; processing time is when the system actually handles that event. They matter because network delays, retries, and out-of-order delivery can cause them to diverge significantly, and computing windowed aggregates based on the wrong notion of time can produce inaccurate or misleading results (e.g., attributing a purchase to the wrong minute).

**6. What is a watermark in stream processing?**
A watermark is a heuristic marker used by a stream processor to estimate how far behind (in event time) it might still receive late-arriving data, letting the system decide when it's safe to finalize and emit the result of a time window despite the possibility of some data still arriving late.

**7. Scenario: You need to generate a weekly sales report aggregating millions of transactions, and it's fine if the report is ready the next morning. Would you use batch or stream processing, and why?**
Batch processing is the better fit here because the freshness requirement is loose (next-morning is acceptable) and the workload benefits from bulk optimization over a large, bounded dataset, making it more cost- and resource-efficient than running continuous stream processing for something that doesn't need sub-second results.

**8. Scenario: You need to detect and block a fraudulent credit card transaction before it's approved. Would you use batch or stream processing, and why?**
Stream processing is required here because the decision must be made within the transaction's brief authorization window (milliseconds to a couple of seconds) — waiting for a nightly batch job would mean the fraudulent transaction has already gone through by the time it's detected.

**9. What is the Lambda architecture, and what problem does it try to solve?**
Lambda architecture runs incoming data through two parallel paths: a batch layer that periodically computes accurate, comprehensive results, and a speed layer that computes fast, approximate real-time results; a serving layer merges both views for queries. It solves the problem of wanting both the accuracy/completeness of batch processing and the low latency of stream processing simultaneously.

**10. What is the main drawback of the Lambda architecture, and how does Kappa architecture address it?**
Lambda's main drawback is that you must build, maintain, and keep in sync two separate codebases implementing similar business logic (one for batch, one for streaming), which is an operational and engineering burden. Kappa architecture addresses this by treating all data — including historical data — as a single replayable stream, using one codebase and simply reprocessing the stream from the beginning when recomputation is needed.

**11. What makes Kappa architecture practical, and what underlying technology does it depend on?**
Kappa architecture is practical because of durable, replayable log-based systems like Apache Kafka, which retain historical events and let consumers reset their offset to reprocess data from any point in time — without such a system, you'd have no way to "replay" history through a stream-only pipeline.

**12. Give a real-world example of a single product using both batch and stream processing for different features, and explain why.**
A music streaming service might use batch processing to generate a weekly personalized playlist (e.g., "Discover Weekly"), since running expensive machine learning models over a month of listening history once a week is efficient and the staleness is acceptable for that feature. The same service might use stream processing to power a live "currently listening" counter or trending-song detection, since those need to reflect activity within seconds, not once a week.
