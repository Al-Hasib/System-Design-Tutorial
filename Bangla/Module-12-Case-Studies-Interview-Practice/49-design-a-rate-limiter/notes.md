# Design a Rate Limiter — Interview Cheat Sheet

[`README.md`](./README.md)-এর দ্রুত-রেফারেন্স সহায়ক। পুরো script পুনরায় না পড়ে interview flow পুনরাবৃত্তি করতে এটি ব্যবহার করুন।

## ১. Requirements

**Functional**
- Per-user limit (authenticated traffic)।
- Per-IP limit (anonymous/unauthenticated traffic)।
- Configurable tier সহ Per-API-key limit (যেমন, free = 100 req/min, paid = 10,000 req/min)।
- Per-endpoint override (যেমন, `/login` এবং `/password-reset` `/search`-এর চেয়ে অনেক কঠোর)।
- একটি `Retry-After` header সহ standard `429 Too Many Requests` response।
- Runtime-এ নিয়মগুলো configurable, redeploy দরকার নেই।

**Non-functional**
- পুরো fleet জুড়ে সঠিকভাবে কার্যকর হতে হবে, per-instance নয় ("distributed enforcement")।
- ন্যূনতম latency যোগ করতে হবে — এটি প্রতিটি request-এর hot path-এ চলে।
- একটি single point of failure হয়ে ওঠা উচিত নয় যা একটি limiter outage-কে একটি সম্পূর্ণ API outage-এ পরিণত করে।
- fair/accurate হতে হবে, কিন্তু perfect precision-এর চেয়ে availability এবং latency বেশি গুরুত্বপূর্ণ।

## ২. Capacity Estimation

| পরিমাণ | অনুমান | নোট |
|---|---|---|
| Peak global request rate | 10M req/sec | পুরো API gateway fleet জুড়ে |
| Gateway instance | ~500 | প্রতি instance-এ ~20K req/sec |
| Exact per-request check-এর জন্য naive Redis ops দরকার | 10M ops/sec | একটি single Redis node সর্বোচ্চ ~100K-200K ops/sec → ৫০-১০০টি shard লাগবে |
| Unique rate-limit key | 50M | Users + API keys + IP buckets |
| প্রতি key memory (sliding window counter) | ~120 bytes | ~24B data + ~60-90B Redis per-key overhead |
| Total counter working set | ~6 GB | সহজে shardable; memory কঠিন constraint নয় — write throughput-ই আসল সমস্যা |

**মূল কথা:** exact, synchronous, প্রতি-request-এক-store-এর-বিরুদ্ধে-check design গুলো এই volume-এ scale করে না — এটাই local pre-aggregation / approximate counting-এর সংখ্যাগত যুক্তি।

## ৩. High-Level Architecture

```
Client → Load Balancer → API Gateway (rate-limiter middleware/sidecar)
                              │
                              ▼
                     Sharded Redis Cluster (counters, Lua scripts)
                              │
                     under limit │ over limit → 429 + Retry-After (short-circuit, never reaches downstream)
                              ▼
                     Downstream Microservices
```

- Reject গুলো edge-এ (gateway) ঘটে, call graph-এর গভীরে নয়।
- ঐচ্ছিক client SDK একটি request পাঠানোর আগেই local optimistic throttling করে।
- ঐচ্ছিক service-mesh sidecar গুলো internally service-to-service limit কার্যকর করে।

## ৪. Key Design Decisions & Trade-offs

### Algorithm তুলনা

| Algorithm | Burst আচরণ | Output smoothness | Memory cost | সেরা ব্যবহার |
|---|---|---|---|---|
| **Fixed Window** | window boundary-তে limit-এর 2x পর্যন্ত অনুমতি দেয় | Choppy, একটি cliff-এ reset হয় | O(1) প্রতি key | সহজ, coarse limit যেখানে boundary burst গ্রহণযোগ্য |
| **Sliding Window Log** | সঠিক, কোনো boundary artifact নেই | Exact | O(n) — প্রতি request একটি entry | কম-volume, উচ্চ-precision limit |
| **Sliding Window Counter** | log-এর কাছাকাছি approximation | মোটামুটি smooth (weighted estimate) | O(1) প্রতি key | ব্যবহারিক default — log-এর precision, fixed window-এর cost |
| **Token Bucket** | bucket capacity পর্যন্ত burst অনুমতি দেয়, তারপর refill rate-এ throttle করে | Bursty তারপর steady | O(1) প্রতি key | Client SDK, API gateway — বৈধ burstiness সহ্য করা (Stripe, AWS API Gateway model) |
| **Leaky Bucket** | কোনো burst pass-through নেই; constant rate-এ queue করে এবং drain করে | কঠোরভাবে uniform | O(1) প্রতি key + queue | ভঙ্গুর downstream system রক্ষা করা যাদের smoothed input দরকার (Nginx `limit_req`) |

### অন্যান্য সিদ্ধান্ত

- **Atomicity:** check-then-increment অবশ্যই একটি atomic Redis Lua `EVAL` হতে হবে, দুটি round trip নয় — বিভিন্ন gateway instance-এ concurrent request জুড়ে race condition এড়ায়।
- **Consistency model:** strict linearizability-র চেয়ে Redis-এর availability/low-latency replication-কে প্রাধান্য দিন — CAP/PACELC-এর সাথে যুক্ত; একটি rate limiter-এর perfect global exactness-এর চেয়ে speed এবং uptime বেশি দরকার।
- **Idempotency:** একটি idempotency key বহনকারী retry করা request গুলোর quota দ্বিগুণ consume করা উচিত নয়।
- **Multi-layer enforcement:** client SDK (optimistic, local) → API gateway (authoritative, global, Redis-backed) → service mesh sidecar (internal service-to-service protection)।
- **Hot keys:** সামগ্রিক cluster sharding নির্বিশেষে একটি single viral user/API key একটি Redis shard-কে saturate করতে পারে — প্রতি request-এ একটি Redis round trip-এর বদলে local pre-aggregation এবং periodic sync দিয়ে প্রতিকার করুন।
- **Clock skew:** per-instance client clock বিশ্বাস করার বদলে Lua script-এর ভেতরে Redis-এর নিজস্ব server clock ব্যবহার করে elapsed time গণনা করুন।
- **Fail-open বনাম fail-closed:** Redis/limiter store-এর call গুলো circuit-break করুন (Circuit Breaker pattern দেখুন)। বেশিরভাগ public read traffic-এর জন্য Fail-open (traffic অনুমতি দেওয়া) নিরাপদ default; security-sensitive endpoint যেমন login-এর জন্য fail-closed (traffic reject করা) নিরাপদ, যেখানে limiter বন্ধ থাকলে একজন attacker উপকৃত হয়।
