# Study Notes: Rate Limiting Algorithms

## তুলনা সারণি (Comparison Table)

| Algorithm | Burst-এর অনুমতি দেয়? | Traffic মসৃণ করে? | Memory Cost | Implementation জটিলতা | সাধারণ Use Case |
|---|---|---|---|---|---|
| Fixed Window Counter | হ্যাঁ, window boundary-তে (limit-এর প্রায় ২ গুণ পর্যন্ত) | না | কম — প্রতি client-এ ১টি counter | কম | সহজ প্রতি-key limit (যেমন, প্রতি API key-তে প্রতি মিনিটে ১০০ request) |
| Sliding Window Log | না (নিখুঁত প্রয়োগ) | কিছুটা (সঠিক window) | বেশি — প্রতি client-এ O(n) timestamp | মাঝারি-উচ্চ | মাঝারি volume-এ নির্ভুলতা-সংবেদনশীল limit |
| Sliding Window Counter | ছোট, সীমাবদ্ধ burst boundary-তে | হ্যাঁ (আনুমানিক) | কম — প্রতি client-এ ২টি counter | মাঝারি | API gateway-এর জন্য ব্যবহারিক ডিফল্ট (log-কে সস্তায় আনুমানিক করে) |
| Token Bucket | হ্যাঁ, bucket capacity পর্যন্ত | না (burst তাৎক্ষণিকভাবে পাস হয়) | কম — প্রতি client-এ token + timestamp | মাঝারি | Burst-প্রবণ বৈধ client সহ্য করা API (Stripe, AWS API Gateway) |
| Leaky Bucket | না (স্থির output rate) | হ্যাঁ (কঠোরভাবে স্থির) | কম-মাঝারি — প্রতি client-এ queue + timestamp | মাঝারি | ভঙ্গুর/fixed-capacity downstream সিস্টেমে traffic shaping (Nginx) |

## মূল সংখ্যা / উদাহরণ (Key Numbers / Examples)

- সাধারণ API rate limit: "প্রতি API key-তে প্রতি মিনিটে ১০০ request" বা "প্রতি user-এ প্রতি ঘণ্টায় ১০০০ request।"
- Stripe: limit endpoint/mode অনুযায়ী ভিন্ন হয়, প্রতি account-এ প্রয়োগ করা হয়, `Retry-After` header সহ `429` ফেরত দেয়।
- AWS API Gateway: কনফিগারযোগ্য steady-state rate (req/sec) + burst capacity (token bucket পরিভাষা সরাসরি উন্মুক্ত)।
- Nginx `limit_req_zone`: `10r/s`-এর মতো একটি rate, একটি ঐচ্ছিক `burst=20` প্যারামিটার সহ যা অতিরিক্ত request গুলোকে তাৎক্ষণিকভাবে প্রত্যাখ্যান না করে queue করে।
- Redis `INCR` + `EXPIRE` প্যাটার্ন: O(1) atomic fixed-window counter; read-check-write atomic রাখতে token bucket / sliding window counter-এর জন্য Lua script (`EVAL`)-এর মাধ্যমে একটি একক round trip প্রয়োজন।

## ইন্টারভিউ পুনরালোচনা — সংক্ষিপ্ত সারসংক্ষেপ (Interview Revision — Bullet Summary)

- Fixed window: সবচেয়ে সস্তা, কিন্তু এর একটি boundary সমস্যা আছে — একটি window edge জুড়ে traffic limit-এর প্রায় ২ গুণ পর্যন্ত burst করতে পারে।
- Sliding window log: নিখুঁত, প্রতিটি request-এর timestamp সংরক্ষণ করে, memory traffic অনুযায়ী বাড়ে — বড় স্কেলে যেমন আছে তেমন খুব কমই ব্যবহৃত হয়।
- Sliding window counter: বর্তমান + পূর্ববর্তী fixed window-এর weighted average — log-এর একটি ব্যবহারিক আনুমানিকরণ, fixed-window-এর মতো memory cost সহ।
- Token bucket: token একটি capacity পর্যন্ত স্থির হারে রিফিল হয়; একটি request একটি token consume করে; নিষ্ক্রিয় token জমা হলে bucket size পর্যন্ত একটি burst-এর অনুমতি দেয়।
- Leaky bucket: request গুলো queue করে এবং একটি কঠোরভাবে স্থির হারে drain হয়; অতিরিক্ত request overflow/প্রত্যাখ্যাত হয়; ইনপুট প্যাটার্ন নির্বিশেষে কখনো leak rate ছাড়িয়ে burst করে না।
- Token bucket বনাম leaky bucket, এক-লাইনের পার্থক্য: token bucket *admission-এর হার* নিয়ন্ত্রণ করে এবং জমানো capacity তাৎক্ষণিকভাবে খরচ করতে দেয় (burst-প্রবণ output); leaky bucket *output-এর হার* নিয়ন্ত্রণ করে এবং সবসময় একটি স্থির হারে মসৃণ করে (non-burst output)।
- Distributed rate limiting-এর instance জুড়ে শেয়ারড state প্রয়োজন — নিখুঁত global প্রয়োগের জন্য atomic `INCR`/Lua script সহ Redis ব্যবহার করুন, অথবা কম latency এবং single point of failure/bottleneck না থাকার জন্য local প্রতি-instance counter দিয়ে eventual consistency মেনে নিন।
- Rate limiting সাধারণত API gateway layer-এ থাকে (API Gateway/BFF ভিডিও দেখুন) এবং প্রায়ই Redis-এর মতো একটি distributed cache-এর উপর নির্ভর করে (Distributed Caching ভিডিও দেখুন) শেয়ারড state store হিসেবে।
