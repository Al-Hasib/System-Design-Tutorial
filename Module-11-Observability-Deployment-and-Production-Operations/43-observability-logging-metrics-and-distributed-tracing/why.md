# Why This Topic Matters: Observability — Logging, Metrics & Distributed Tracing

> **In one sentence:** In a monolith you attach a debugger; in a distributed system the only thing you have is what the system told you about itself while it was running, so anything it did not record is permanently unknowable.

## The World Before This Idea

3 a.m. A customer reports that checkout is failing intermittently — perhaps one request in twenty. Your system has twelve services.

You SSH into a box and tail a log. It scrolls past at thousands of lines a second, interleaved from many concurrent requests, with no way to isolate one user's journey. You do not know which of the twelve services is at fault, so you repeat this eleven more times. The dashboard shows average latency of 120 ms — healthy — because the 5% of requests taking 8 seconds are invisible in an average. Nothing tells you whether this started an hour ago or last Tuesday.

You end up guessing, restarting a service, and watching to see if the symptom stops. Sometimes it does, which teaches you nothing and leaves the cause in place.

## The Problems It Solves

### 1. No way to follow one request across services
**What you see:** A slow or failing request, and twelve separate log streams with no connection between them.

**Why it happens:** Each service logs independently. There is nothing tying the log line in service A to the one in service G that came from the same user action.

**How distributed tracing solves it:** A trace ID is generated at the edge and propagated through every downstream call, with each service recording spans under it. You get one view showing the complete path, with the time spent in each hop. "The request took 8 seconds and 7.6 of them were in the inventory service waiting on a database query" is an answer you cannot get any other way. In a microservices architecture, tracing is not a nice-to-have — it is the replacement for the stack trace you lost when you split the monolith.

### 2. Averages that hide the problem
**What you see:** Dashboards that look fine while users are complaining.

**Why it happens:** An average is dominated by the common case. If 95% of requests take 50 ms and 5% take 10 seconds, the average is 545 ms — a number that describes no actual request and conceals the fact that one user in twenty is having an unusable experience.

**How percentile metrics solve it:** p50, p95, p99, and p99.9 show the shape of the distribution. The p99 is where your worst-served users live, and at scale those are a lot of people — one in a hundred requests, and far more than one in a hundred *sessions*, since a session makes many requests. Tracking tail latency rather than averages is one of the highest-value changes a team can make.

### 3. No way to tell normal from abnormal
**What you see:** An error rate of 0.3%. Is that fine? Nobody knows, because nobody knows what it was last week.

**Why it happens:** Without retained time-series data there is no baseline, so every number is uninterpretable and every incident investigation starts from zero.

**How metrics solve it:** Cheap, continuously collected numeric series with history let you compare to yesterday, last week, and the last deploy. This is also what makes alerting possible — alerts fire on deviation from a known baseline, and on symptoms users actually feel, not on every transient blip.

### 4. Logs that cannot be searched or correlated
**What you see:** Free-text log lines like `Error processing order`, with no order ID, no user ID, no trace ID, and no consistent format.

**Why it happens:** Logging written for a human reading a terminal, not for a machine querying billions of lines.

**How structured logging solves it:** Emit key-value records (JSON) with consistent field names, always including the trace ID, and ship them to a central store. Now you can query — "all errors for user 4821 in the last hour across all services" — instead of grepping twelve machines. The trace ID is the join key that makes logs, metrics, and traces one system instead of three.

## The Price You Pay

- **It is genuinely expensive.** Observability data frequently costs more than the infrastructure it observes. Vendor bills for log ingestion and retention are a common budget surprise, and the mitigations — sampling traces, aggressive retention tiers, cardinality limits on metrics — each trade away some ability to answer questions later.
- **Instrumentation is real engineering work.** Context propagation across threads, async boundaries, and queue hops is fiddly and easy to break silently. Auto-instrumentation helps and does not cover everything.
- **Performance overhead.** Tracing every request costs CPU and network, which is why sampling exists — and why the request you most want to investigate may be the one that was not sampled. Tail-based sampling (decide after seeing the outcome) helps, at a higher cost.
- **High-cardinality metrics explode.** A metric labeled with user ID creates a series per user and can take down your metrics backend. Cardinality discipline is a recurring operational lesson.
- **Logs leak secrets and personal data.** Request bodies, headers, and tokens end up in a searchable store accessible to a wide audience. This is a real and common compliance failure.
- **Alert fatigue destroys the value.** Too many alerts, or alerts on causes rather than symptoms, train people to ignore the pager. A noisy alerting setup is worse than none.

## When You Need It — and When You Don't

| Invest heavily when | Basics are enough when |
|---|---|
| You run microservices — tracing is mandatory | A single service with a single database |
| You have a real SLA or paying customers | An internal tool with tolerant users |
| Incidents are expensive or frequent | Pre-launch, no traffic |
| The system is large enough that no one person understands it all | — |

Even at the low end, three things are always worth it: **structured logs with a request ID, a handful of golden-signal metrics (latency, traffic, errors, saturation), and alerts on user-visible symptoms.**

## Why This Shows Up in Interviews

Observability is the part of a design that separates people who have built systems from people who have designed them on paper. It is rarely the main question and is frequently the closing one: "how would you know this was broken?" or "how would you debug a slow request here?" Candidates who volunteer tracing with correlation IDs, percentile-based SLOs rather than averages, and symptom-based alerting demonstrate operational maturity in about thirty seconds. In a microservices design, failing to mention tracing at all is a noticeable gap.

## How It Connects

Observability becomes mandatory the moment you adopt **microservices** (topic 30) and **event-driven architecture** (topic 22), because both destroy the local stack trace. It is what lets you tune **circuit breakers and timeouts** (topic 26) with real latency data, verify **cache hit rates** (topic 17), monitor **replication lag** (topic 13) and consumer lag (topic 20), and detect **compaction**-driven latency spikes (topic 38). It is the prerequisite for safe **zero-downtime deploys** (topic 45) and for **chaos engineering** (topic 46) — you cannot run an experiment you cannot measure.

**Next:** [Containers & Orchestration](../44-containers-and-orchestration-docker-and-kubernetes-fundamentals/why.md) — how all these services get packaged, scheduled, and kept running.
