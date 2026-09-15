# Why This Topic Matters: Batch Processing vs Stream Processing

> **In one sentence:** Some answers are worth waiting until tomorrow for, and some are worthless if they arrive a minute late — choosing the wrong processing model means either burning money on infrastructure you don't need or shipping insights that arrive after they mattered.

## The World Before This Idea

Two opposite failure modes, both common:

**Batch where you needed streaming.** Fraud detection runs as a nightly job. A stolen card is used at 9 a.m. and flagged at 2 a.m. the next morning — after seventeen hours of fraudulent purchases. The model is accurate and the pipeline is well-engineered. It is also useless, because the answer arrives long after the only moment it could have been acted on.

**Streaming where you needed batch.** A team builds a real-time pipeline to compute monthly financial reports. It runs continuously, costs ten times a nightly job, is far harder to debug, and produces numbers that are re-stated whenever a late event arrives — for a report nobody reads until the fifth of the month.

Both teams were competent. Both mismatched the processing model to the *decision latency* the business actually required.

## The Problems It Solves

### 1. Insight that arrives after the moment to act has passed
**What you see:** Alerts, recommendations, and detections that are technically correct and practically irrelevant.

**Why it happens:** Batch jobs have a floor on latency equal to their schedule interval, plus their runtime. A daily job can never tell you anything in under 24 hours.

**How stream processing solves it:** Events are processed as they arrive, so results are available within seconds. For fraud, abuse detection, live dashboards, dynamic pricing, and trending content, this is the difference between a system that works and one that doesn't.

### 2. Reprocessing that takes a week
**What you see:** You find a bug in a metric computation. Fixing it means recomputing two years of history, and there's no mechanism to do it.

**Why it happens:** Streaming systems compute incrementally over a window and often don't retain raw inputs.

**How batch solves it:** Batch operates over a complete, bounded, stored dataset. Fix the code, re-run the job, get corrected results — with full access to all the data at once, which also enables computations (global sorts, full joins, model training) that streaming simply cannot do efficiently.

### 3. Correctness problems caused by time itself
**What you see:** A mobile app buffers events offline and uploads them three hours later. Your hourly counts for that window are wrong, and they were already published.

**Why it happens:** In a stream, the time an event *occurred* and the time it *arrived* differ, sometimes by a lot. Any windowed aggregate has to decide how long to wait for stragglers.

**How this topic solves it:** It makes event time versus processing time, watermarks, and late-arrival handling explicit design decisions rather than surprises. Batch sidesteps the issue by waiting for the window to be closed before computing — which is part of why it's simpler and part of why it's slower.

### 4. Paying streaming prices for batch problems
**What you see:** An always-on cluster running 24/7 to produce numbers consumed once a day.

**Why it happens:** "Real-time" sounds strictly better, so it gets chosen by default.

**How this topic solves it:** It reframes the choice around a single question — *how quickly does someone need to act on this?* — which usually reveals that a large fraction of data work is genuinely fine as a scheduled job at a fraction of the cost and complexity.

## The Price You Pay

**Streaming costs:**
- Always-on infrastructure, and the operational burden that comes with it.
- Stateful operations (windows, joins, aggregations) require managed state, checkpointing, and recovery — this is where most of the real complexity lives.
- Exactly-once semantics are achievable but demanding; at-least-once plus idempotency is the pragmatic norm.
- Debugging an unbounded stream is genuinely harder than debugging a job you can re-run on a fixed input.
- Backpressure: if producers outpace consumers, something must give.

**Batch costs:**
- Latency equal to the schedule, always.
- Bursty resource usage — idle for hours, then a massive spike.
- A failed job can mean a whole day with no data, and re-running may not be possible before the next window.
- Results are stale by construction, and users have to understand that.

**And the hybrid has costs too.** Lambda architecture (running both a batch and a streaming path) means maintaining the same business logic twice, in two systems, and reconciling their disagreements. Kappa architecture (streaming only, replayed from the log for reprocessing) avoids the duplication but requires a durable, replayable log and the discipline to keep everything reprocessable.

## When You Need It — and When You Don't

| Stream when | Batch when |
|---|---|
| Decisions must be made in seconds (fraud, abuse, alerting) | Results are consumed daily or less often |
| Users see live-updating values | You need global operations over the full dataset |
| Data is continuous and unbounded | You need to reprocess history after a fix |
| Late data is worse than approximate data | Accuracy and completeness beat freshness |
| — | Cost efficiency matters and latency doesn't |

## Why This Shows Up in Interviews

Analytics, metrics, recommendations, and trending features appear in most large design prompts. The signal interviewers want is that you ask about *decision latency* before choosing an architecture — "how fresh does this number need to be?" — and then pick accordingly rather than defaulting to real-time because it sounds impressive. Candidates who can also name the hard parts of streaming (event time versus processing time, windowing, late arrivals, state management) are demonstrating depth that goes well beyond naming Kafka and Flink.

## How It Connects

Both models consume from the same **message queue** infrastructure (topic 20) and the **events** produced by an event-driven system (topic 22). Streaming systems lean heavily on **probabilistic data structures** (topic 42) to approximate counts and cardinalities in bounded memory, and on **idempotency** (topic 29) to survive redelivery. The freshness-versus-accuracy trade is another instance of the **consistency** trade-offs from Module 3, and **logical clocks** (topic 41) underpin reasoning about event ordering.

**Next:** [Consistent Hashing Explained](../../Module-06-Distributed-Systems-Concepts/24-consistent-hashing-explained/why.md) — how to spread data across nodes without reshuffling everything each time the cluster changes.
