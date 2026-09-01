# Study Notes: Observability

## Definitions

- **Observability:** The property of being able to investigate and understand a system's internal state from the outside, including failure modes not anticipated in advance.
- **Monitoring:** Watching a predefined set of known failure modes/metrics and alerting when they cross a threshold.
- **Structured logging:** Logging discrete events as structured records (e.g., JSON) with consistent fields (timestamp, service, request ID, severity), rather than free-text sentences.
- **Metric:** A numeric measurement aggregated over time (e.g., requests/sec, p99 latency, error rate).
- **Distributed tracing:** Tracking a single request's path across multiple services using a shared trace ID, broken into linked spans.
- **Span:** One unit of work within a trace — a start time, duration, and metadata for one service's handling of its piece of a request.
- **Alert fatigue:** So many low-signal alerts that engineers start ignoring all of them, including ones that matter.

## The Three Pillars

| Pillar | Answers | Granularity | Example tools |
|---|---|---|---|
| Logs | What exactly happened, in this one event? | Very high (per-event) | ELK stack (Elasticsearch, Logstash, Kibana), Loki |
| Metrics | How is the system behaving, in aggregate, over time? | Low (pre-aggregated) | Prometheus, Grafana, Datadog |
| Traces | What was this specific request's full path, and where did it slow down/fail? | Per-request, cross-service | Jaeger, Zipkin, OpenTelemetry |

## Monitoring vs. Observability

| | Monitoring | Observability |
|---|---|---|
| Scope | Predefined, known failure modes | Any failure mode, including unanticipated ones |
| Question answered | "Is X (a thing I already thought to watch) OK?" | "What actually happened, and why?" |
| Requires | Dashboards/alerts for known metrics | Rich structured logs + high-cardinality metrics + tracing |

## Good Alerting Practice

- Alert on **symptoms** that affect users (elevated error rate, breached latency SLO), not every possible internal cause.
- Too many low-signal alerts → alert fatigue → real incidents get missed among the noise.
- Pair alerts with runbooks/dashboards that let the on-call engineer immediately start narrowing down the cause.

## How Distributed Tracing Works

1. A trace ID is generated when a request first enters the system (e.g., at the API gateway/load balancer).
2. That trace ID is propagated via a header (e.g., in HTTP or gRPC metadata) to every downstream service call.
3. Each service records its own span (start time, duration, metadata) tagged with that trace ID.
4. Spans are linked into a parent-child tree matching the actual call graph, viewable as a timeline in a tracing UI (Jaeger/Zipkin).

## Key Numbers / Facts

- OpenTelemetry is the current industry-standard, vendor-neutral framework/spec for generating and exporting traces, metrics, and logs.
- p99 latency (99th percentile) is the standard metric for "how slow is it for the worst 1% of requests" — often what actually matters for user experience, more than an average.

## Summary

- Logs, metrics, and traces each answer a different question — none of them alone gives you full visibility into a distributed system's behavior.
- Observability (investigating unknown failure modes) is a broader, stronger property than monitoring (watching known ones).
- Distributed tracing is specifically what makes multi-service request paths debuggable, by propagating one trace ID across every service boundary a request crosses.
