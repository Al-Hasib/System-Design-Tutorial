# Diagrams: Observability

## 1. The Three Pillars and What Each Answers

```mermaid
flowchart TB
    Event[Something happens<br/>in production] --> Logs["Logs:<br/>What exactly happened, here?"]
    Event --> Metrics["Metrics:<br/>How is the system doing, in aggregate?"]
    Event --> Traces["Traces:<br/>What was this request's full path?"]

    Logs --> Agg["Log aggregation<br/>(ELK, Loki)"]
    Metrics --> Dash["Dashboards + Alerting<br/>(Prometheus, Grafana)"]
    Traces --> TraceUI["Trace visualization<br/>(Jaeger, Zipkin)"]
```
*Each pillar captures a different dimension of system behavior — none of them alone gives a complete picture of a distributed system.*

## 2. Distributed Trace Across Microservices

```mermaid
sequenceDiagram
    participant Client
    participant GW as API Gateway
    participant Checkout as Checkout Service
    participant Inventory as Inventory Service
    participant Payment as Payment Service

    Client->>GW: Request (trace ID generated: abc-123)
    GW->>Checkout: Forward (trace ID: abc-123)
    Checkout->>Inventory: Check stock (trace ID: abc-123, span 2)
    Inventory-->>Checkout: OK (span 2 done, 15ms)
    Checkout->>Payment: Charge card (trace ID: abc-123, span 3)
    Payment-->>Checkout: OK (span 3 done, 3800ms - slow!)
    Checkout-->>GW: Response
    GW-->>Client: Response
```
*The same trace ID (abc-123) is propagated through every service call, letting a tracing tool reconstruct the full path and immediately show that the Payment Service's span accounted for nearly all of the request's latency.*

## 3. Alerting on Symptoms, Not Every Internal Cause

```mermaid
flowchart LR
    subgraph Causes["Many possible internal causes"]
        C1[DB connection pool exhausted]
        C2[Downstream service slow]
        C3[GC pause]
        C4[Disk nearly full]
    end

    Causes --> Symptom["User-facing symptom:<br/>p99 latency exceeds 1s"]
    Symptom --> Alert[Single, high-signal alert fires]
    Alert --> OnCall[On-call engineer paged]
    OnCall --> Investigate["Investigate using logs/metrics/traces<br/>to find the actual root cause"]
```
*Alerting directly on user-facing symptoms (not every possible internal cause) keeps the signal-to-noise ratio high — the actual root cause is found afterward, during investigation, using the other observability pillars.*
