# অধ্যয়ন নোট (Study Notes): Observability

## সংজ্ঞা (Definitions)

- **Observability:** বাইরে থেকে একটি সিস্টেমের অভ্যন্তরীণ অবস্থা তদন্ত এবং বোঝার সক্ষমতা, যার মধ্যে আগে থেকে অনুমান করা হয়নি এমন failure mode-ও অন্তর্ভুক্ত।
- **Monitoring:** পরিচিত failure mode/metric-এর একটি পূর্বনির্ধারিত সেট পর্যবেক্ষণ করা এবং সেগুলো একটি threshold অতিক্রম করলে alert করা।
- **Structured logging:** ফ্রি-টেক্সট বাক্যের বদলে সামঞ্জস্যপূর্ণ field (timestamp, service, request ID, severity) সহ structured record (যেমন JSON) হিসেবে স্বতন্ত্র ঘটনা log করা।
- **Metric:** সময়ের সাথে সমষ্টিকৃত একটি সাংখ্যিক পরিমাপ (যেমন, requests/sec, p99 latency, error rate)।
- **Distributed tracing:** একটি shared trace ID ব্যবহার করে একাধিক service জুড়ে একটি একক request-এর পথ ট্র্যাক করা, যুক্ত span-এ বিভক্ত।
- **Span:** একটি trace-এর মধ্যে কাজের একটি একক — একটি service-এর একটি request-এর নিজের অংশ পরিচালনার জন্য একটি start time, duration, এবং metadata।
- **Alert fatigue:** এত বেশি low-signal alert যে ইঞ্জিনিয়াররা সবগুলোকে উপেক্ষা করা শুরু করে, এমনকি যেগুলো গুরুত্বপূর্ণ সেগুলোকেও।

## তিনটি স্তম্ভ (The Three Pillars)

| স্তম্ভ | উত্তর দেয় | Granularity | উদাহরণ টুল |
|---|---|---|---|
| Logs | ঠিক কী ঘটেছে, এই একটি ঘটনায়? | খুব উচ্চ (per-event) | ELK stack (Elasticsearch, Logstash, Kibana), Loki |
| Metrics | সিস্টেম কেমন আচরণ করছে, সামষ্টিকভাবে, সময়ের সাথে? | নিম্ন (আগে থেকে সমষ্টিকৃত) | Prometheus, Grafana, Datadog |
| Traces | এই নির্দিষ্ট request-এর সম্পূর্ণ পথ কী ছিল, এবং এটা কোথায় ধীর হয়েছে/ব্যর্থ হয়েছে? | Per-request, cross-service | Jaeger, Zipkin, OpenTelemetry |

## Monitoring বনাম Observability

| | Monitoring | Observability |
|---|---|---|
| পরিধি | পূর্বনির্ধারিত, পরিচিত failure mode | যেকোনো failure mode, অপ্রত্যাশিতগুলোসহ |
| উত্তরিত প্রশ্ন | "X (এমন কিছু যা আমি ইতিমধ্যে দেখার কথা ভেবেছি) কি ঠিক আছে?" | "আসলে কী ঘটেছে, এবং কেন?" |
| প্রয়োজন | পরিচিত metric-এর জন্য dashboard/alert | সমৃদ্ধ structured logs + high-cardinality metrics + tracing |

## ভালো Alerting চর্চা

- ব্যবহারকারীদের প্রভাবিত করে এমন **symptom**-এর উপর alert করুন (elevated error rate, breached latency SLO), প্রতিটি সম্ভাব্য অভ্যন্তরীণ কারণে নয়।
- অনেক বেশি low-signal alert → alert fatigue → শব্দের মধ্যে প্রকৃত incident হারিয়ে যায়।
- Alert-কে runbook/dashboard-এর সাথে জোড়া দিন যাতে on-call ইঞ্জিনিয়ার অবিলম্বে কারণ চিহ্নিত করা শুরু করতে পারে।

## Distributed Tracing কীভাবে কাজ করে

1. একটি request প্রথম সিস্টেমে প্রবেশ করলে (যেমন, API gateway/load balancer-এ) একটি trace ID তৈরি হয়।
2. সেই trace ID একটি header-এর মাধ্যমে (যেমন, HTTP বা gRPC metadata-এ) প্রতিটি downstream service call-এ প্রচারিত হয়।
3. প্রতিটি service সেই trace ID দিয়ে ট্যাগ করা নিজস্ব span (start time, duration, metadata) রেকর্ড করে।
4. Span গুলো প্রকৃত call graph-এর সাথে মিলিয়ে একটি parent-child গাছে যুক্ত হয়, যা একটি tracing UI-তে (Jaeger/Zipkin) একটি timeline হিসেবে দেখা যায়।

## গুরুত্বপূর্ণ সংখ্যা / তথ্য (Key Numbers / Facts)

- OpenTelemetry হলো trace, metrics, এবং logs তৈরি এবং export করার জন্য বর্তমান industry-standard, vendor-neutral framework/spec।
- p99 latency (99th percentile) হলো "সবচেয়ে খারাপ 1% request-এর জন্য এটা কতটা ধীর" — এর মানসম্মত metric, যা প্রায়ই গড়ের চেয়ে বেশি ব্যবহারকারীর অভিজ্ঞতার জন্য আসলে গুরুত্বপূর্ণ।

## সারসংক্ষেপ (Summary)

- Logs, metrics, এবং traces প্রত্যেকে একটি ভিন্ন প্রশ্নের উত্তর দেয় — এগুলোর কোনোটাই একা একটি distributed সিস্টেমের আচরণে সম্পূর্ণ দৃশ্যমানতা দেয় না।
- Observability (অজানা failure mode তদন্ত করা) হলো একটি বৃহত্তর, শক্তিশালী বৈশিষ্ট্য monitoring (পরিচিতগুলো পর্যবেক্ষণ করা)-এর চেয়ে।
- Distributed tracing সুনির্দিষ্টভাবে সেটাই যা multi-service request path-কে debug করার যোগ্য করে তোলে, একটি request যে প্রতিটি service boundary অতিক্রম করে তার মধ্য দিয়ে একটি trace ID প্রচার করে।
