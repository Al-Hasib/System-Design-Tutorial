# Observability: Logging, Metrics & Distributed Tracing

**Difficulty:** Intermediate
**Estimated length:** 14-18 min
**Prerequisites:** [07 - Load Balancing Explained](../../Module-02-Networking-and-Communication/07-load-balancing-explained/README.md), [30 - Monolith vs Microservices](../../Module-07-Architecture-Patterns/30-monolith-vs-microservices/README.md)

## Learning Objectives

- Explain what observability means and why it's distinct from simply "having logs."
- Describe the three pillars of observability — logs, metrics, and traces — and what question each one is best at answering.
- Explain how distributed tracing propagates a single trace ID across service boundaries to reconstruct a request's full path.
- Distinguish monitoring (watching known failure modes) from observability (being able to investigate unknown ones).
- Design a basic alerting strategy that avoids both missed incidents and alert fatigue.

## Script

### Hook / Intro

Every system we've designed in this course so far has been drawn as boxes and arrows on a whiteboard. In production, none of that structure is visible to you by default — a request comes in, disappears into your system, and either a response comes out or it doesn't, and if something goes wrong, you have exactly the information you specifically decided in advance to capture, and not one byte more. This is observability: the practice of building a system so that when something goes wrong — especially something you never anticipated — you can actually figure out what happened, from the outside, without attaching a debugger to a live production process. Today we cover the three pillars that make this possible, and why microservices architectures (recall Module 7) make this dramatically harder than it looks.

### Logs: What Happened, Here, Right Now

A **log** is a timestamped, discrete record of a specific event: "user 5 logged in," "payment failed with error X," "cache miss for key Y." Logs are the most granular, detailed pillar — they can tell you exactly what happened at a specific point in one specific process. The practical challenge at scale isn't writing logs — every engineer already does that — it's making them useful once you have millions of them, scattered across hundreds of service instances. This is why production systems universally move toward **structured logging**: instead of a free-text sentence, each log entry is a structured record (commonly JSON) with consistent fields — timestamp, service name, request ID, severity level, and whatever specific context matters — shipped to a centralized log aggregation system (like the ELK stack — Elasticsearch, Logstash, Kibana — or a managed equivalent) where they can actually be searched, filtered, and correlated, instead of living as unsearchable text files scattered across servers you'd have to individually SSH into.

### Metrics: How Is the System Doing, Over Time?

A **metric** is a numeric measurement aggregated over time — requests per second, p99 latency, error rate, CPU utilization, queue depth. Where a log tells you about one specific event, a metric tells you about the *shape* of behavior across thousands or millions of events, cheaply, because it's pre-aggregated rather than storing every individual data point. This is what powers dashboards and, critically, **alerting**: you don't want a human watching a dashboard 24/7, you want a system that pages someone the instant error rate crosses a threshold or latency degrades past an agreed bound. The standard toolchain here is something like Prometheus (which "scrapes" metrics from services on a regular interval and stores them as time series) paired with Grafana for visualization — and a well-designed alerting strategy pages on **symptoms** that actually affect users (elevated error rate, breached latency SLOs) rather than every possible internal cause, which is what prevents the very real, very common failure mode of **alert fatigue**: so many low-signal pages that engineers start ignoring all of them, including the one that matters.

### Distributed Tracing: What Happened to *This* Request, Across Every Service It Touched?

Here's where microservices architectures (Module 7) genuinely change the game. In a monolith, a slow request is one process you can profile directly. In a microservices architecture, a single user-facing request might fan out across a dozen internal service calls — logs and metrics alone, gathered independently per service, don't tell you which of those dozen calls was actually the slow one, or where a specific request failed along its path. **Distributed tracing** solves this by generating a unique **trace ID** the moment a request enters the system, and propagating that same trace ID through every downstream service call the request touches — typically as an HTTP header (recall gRPC/HTTP from Module 8) that each service reads, logs alongside its own work, and forwards to whatever it calls next. Each individual unit of work within that trace (one service's handling of its piece of the request) is recorded as a **span**, with a start time, duration, and its own metadata, and spans are linked into a parent-child tree matching the actual call graph. Tools like Jaeger, Zipkin, or a managed equivalent then let you pull up one specific slow or failed request and see, visually, exactly which of the dozen services it touched, in what order, and precisely which one took 800ms while the others took 5ms each. This transforms "the checkout flow is slow sometimes" from a mystery into a five-minute investigation.

### Monitoring vs. Observability

It's worth being precise about a distinction that gets blurred constantly: **monitoring** is watching a predefined set of known failure modes — "alert if CPU exceeds 90%," "alert if the health check fails." It answers questions you thought to ask in advance. **Observability** is the broader property of being able to ask *new* questions about your system's internal state after something unexpected happens, without having shipped new code to add the specific instrumentation you now realize you need. A system can be heavily monitored (dozens of dashboards and alerts) and still not be observable, if none of those dashboards happen to capture the specific, novel failure mode that just occurred. Rich structured logs, granular metrics with useful dimensions (not just "error rate" but "error rate by endpoint, by region, by client version"), and distributed tracing together are what make a system observable — genuinely investigable after the fact, for failure modes nobody specifically anticipated.

### Real-World Example

Imagine an on-call engineer gets paged: checkout latency's p99 has spiked from 200ms to 4 seconds. The metrics dashboard (Prometheus/Grafana) confirms the spike and narrows it to the checkout service specifically, but not *why*. Pulling up a slow trace in Jaeger for one of the affected requests shows the full call graph: API gateway → checkout service → inventory service → payment service → notification service, with span durations attached to each hop — and it's immediately visible that the payment service call alone accounts for 3.8 of those 4 seconds, while everything else is normal. Now the investigation is scoped: pull up the payment service's structured logs, filtered to that exact trace ID, and find the specific error or slow downstream dependency inside payment processing — going from "checkout is slow" to "this specific downstream call in the payment service" in minutes, not hours of guessing.

### Recap

Observability means being able to investigate what actually happened in your system, including failure modes you never specifically anticipated — not just watching a predefined set of dashboards. Logs capture detailed, individual events; metrics capture the aggregated shape of behavior over time and power alerting; distributed tracing propagates a trace ID across every service a request touches, letting you reconstruct exactly which hop in a distributed call graph was actually responsible for a slow or failed request. Monitoring (watching known failure modes) is necessary but not sufficient — true observability requires all three pillars working together, especially once a system is decomposed into more than one service.

### What's Next

We've covered how to see what a running system is actually doing. Next video looks at how that system is actually packaged and scheduled to run in the first place — containers and orchestration, the infrastructure layer underneath every "just deploy it" assumption we've made so far in this course.

## Key Takeaways

- Observability means being able to investigate unknown/unanticipated failure modes after the fact, not just watching a predefined set of dashboards — that's monitoring, which is necessary but not sufficient on its own.
- Logs are detailed, per-event records; structured logging (JSON, consistent fields) shipped to a centralized aggregation system is what makes them searchable at scale.
- Metrics are pre-aggregated numeric measurements over time (latency, error rate, throughput) that power dashboards and alerting — alert on user-facing symptoms to avoid alert fatigue.
- Distributed tracing propagates a trace ID across every service a request touches, breaking the request into linked spans so you can see exactly which hop in a distributed call graph caused a slowdown or failure.
- Rich structured logs, high-cardinality metrics, and distributed tracing together — not any one alone — are what make a distributed system genuinely observable.
