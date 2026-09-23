# ডায়াগ্রাম (Diagrams): Observability

## ১. তিনটি স্তম্ভ এবং প্রতিটি যা উত্তর দেয়

```mermaid
flowchart TB
    Event[Something happens<br/>in production] --> Logs["Logs:<br/>What exactly happened, here?"]
    Event --> Metrics["Metrics:<br/>How is the system doing, in aggregate?"]
    Event --> Traces["Traces:<br/>What was this request's full path?"]

    Logs --> Agg["Log aggregation<br/>(ELK, Loki)"]
    Metrics --> Dash["Dashboards + Alerting<br/>(Prometheus, Grafana)"]
    Traces --> TraceUI["Trace visualization<br/>(Jaeger, Zipkin)"]
```

*প্রতিটি স্তম্ভ সিস্টেম আচরণের একটি ভিন্ন মাত্রা ক্যাপচার করে — এগুলোর কোনোটাই একা একটি distributed সিস্টেমের সম্পূর্ণ ছবি দেয় না।*

## ২. Microservices জুড়ে একটি Distributed Trace

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

*একই trace ID (abc-123) প্রতিটি service call-এর মধ্য দিয়ে প্রচারিত হয়, যা একটি tracing টুলকে সম্পূর্ণ পথ পুনর্গঠন করতে এবং সাথে সাথে দেখাতে দেয় যে Payment Service-এর span-ই request-এর latency-র প্রায় সবটুকুর জন্য দায়ী ছিল।*

## ৩. প্রতিটি অভ্যন্তরীণ কারণে নয়, Symptom-এর উপর Alerting

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

*User-facing symptom-এর উপর সরাসরি alert করা (প্রতিটি সম্ভাব্য অভ্যন্তরীণ কারণের উপর নয়) signal-to-noise অনুপাত উচ্চ রাখে — প্রকৃত root cause পরে খুঁজে বের করা হয়, তদন্তের সময়, বাকি observability স্তম্ভগুলো ব্যবহার করে।*
