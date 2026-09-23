# আরও পড়াশোনা ও রেফারেন্স

## Official Docs

- [Amazon DynamoDB — Read Consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html) — eventually consistent বনাম strongly consistent read সম্পর্কে official documentation।
- [Stripe API — Idempotent Requests](https://stripe.com/docs/api/idempotent_requests) — API request নিরাপদে retry করার জন্য idempotency key ব্যবহার সম্পর্কিত official documentation।

## Papers

- Werner Vogels, ["Eventually Consistent"](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html) — Amazon-এর CTO-র লেখা eventual consistency-এর একটি ব্যাপকভাবে উদ্ধৃত ব্যাখ্যা, মূলত ACM Queue-তে প্রকাশিত।

## আরও পড়াশোনা

- Wikipedia, ["Consistency model"](https://en.wikipedia.org/wiki/Consistency_model) — linearizability, sequential, এবং causal consistency-সহ consistency model-এর সম্পূর্ণ spectrum-এর একটি overview।
- Wikipedia, ["Eventual consistency"](https://en.wikipedia.org/wiki/Eventual_consistency) — background এবং formal সংজ্ঞা।
- Wikipedia, ["Linearizability"](https://en.wikipedia.org/wiki/Linearizability) — strong consistency model-এর formal সংজ্ঞা এবং ইতিহাস।
- [Jepsen.io](https://jepsen.io/) — network partition এবং fault-এর অধীনে বাস্তব distributed database-গুলোর প্রকৃত consistency guarantee-এর স্বাধীন বিশ্লেষণ।
