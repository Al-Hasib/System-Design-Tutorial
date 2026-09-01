# Practice & Interview Questions

**1. What's the difference between monitoring and observability?**
Monitoring is watching a predefined set of known failure modes/metrics and alerting when they cross a threshold — it answers questions you thought to ask in advance. Observability is the broader ability to investigate and understand a system's internal state after something unexpected happens, including failure modes nobody specifically anticipated, without needing to ship new code first.

**2. Why do production systems move toward structured logging (e.g., JSON) instead of free-text log lines?**
At scale, logs are scattered across many service instances and number in the millions. Structured logs with consistent fields (timestamp, service, request ID, severity) can be indexed, searched, and filtered systematically in a centralized aggregation system — free-text logs are much harder to search and correlate across services at that volume.

**3. What question does a metric answer that a log can't, and vice versa?**
A metric answers "how is the system behaving in aggregate, over time?" (e.g., p99 latency, error rate) cheaply, because it's pre-aggregated rather than storing every individual event. A log answers "what exactly happened in this one specific event?" with full detail, but isn't practical to scan in aggregate the way a metric is.

**4. Explain how a trace ID enables distributed tracing across microservices.**
A unique trace ID is generated when a request first enters the system and is propagated (typically via a header) to every downstream service call that request triggers. Each service records its own span — tagged with that same trace ID — for its piece of the work, and a tracing tool links all the spans from one trace ID into a single timeline showing the request's full path across every service it touched.

**5. What is a "span" in distributed tracing?**
A span is one unit of work within a trace — typically one service's handling of its part of a request — recorded with a start time, duration, and metadata, and linked to its parent/child spans to reconstruct the full call graph.

**6. Why is alerting on user-facing symptoms (like elevated error rate or breached latency SLOs) generally better than alerting on every possible internal cause?**
Alerting on every possible internal cause (e.g., every metric that could theoretically indicate a problem) produces a high volume of low-signal alerts, leading to alert fatigue — engineers start ignoring alerts, including the ones that matter. Alerting on symptoms that actually affect users keeps the signal-to-noise ratio high; the specific internal root cause is then found during investigation using logs, metrics, and traces.

**7. Scenario: A single user-facing request in a microservices architecture fans out to 8 internal services, and it's intermittently slow. Why are per-service logs and metrics, gathered independently, insufficient to diagnose this?**
Per-service logs and metrics show each service's own behavior in isolation, but don't tell you how these 8 calls relate to one specific slow request, or which one of the 8 was actually the bottleneck for that particular occurrence. Distributed tracing is needed to tie all 8 services' handling of that one request together via a shared trace ID and see exactly which span accounted for the latency.

**8. Can a system be heavily monitored (many dashboards and alerts) and still not be observable? Explain.**
Yes — a system can have dozens of dashboards and alerts covering every known failure mode the team anticipated, yet still lack observability if none of those dashboards happen to capture a novel, unanticipated failure mode. Observability requires rich enough structured logs, high-cardinality metrics, and tracing to investigate something new after the fact, not just watch a fixed, predefined set of indicators.

**9. Why is p99 latency often a more useful metric to alert on than average latency?**
Average latency can look healthy even while a meaningful fraction of users experience much worse performance, because a few very fast requests can offset many slow ones in an average. p99 latency specifically captures the experience of the worst 1% of requests, which better reflects real user-facing pain that an average would mask.

**10. True or False: Adding distributed tracing to a system removes the need for logs and metrics.**
False. Logs, metrics, and traces each answer a different question and are complementary, not substitutes for each other. Tracing tells you a request's path and where time was spent; you'd still use logs to see the specific error/detail within the slow span, and metrics to detect the aggregate symptom (like a latency spike) that prompted investigating a trace in the first place.
