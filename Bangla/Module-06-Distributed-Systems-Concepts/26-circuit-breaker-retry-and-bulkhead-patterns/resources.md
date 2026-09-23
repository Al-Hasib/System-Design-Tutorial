# আরও পড়ার জন্য ও রেফারেন্স

## অফিসিয়াল ডকুমেন্টেশন

- [Resilience4j Documentation](https://resilience4j.readme.io/docs) — JVM-এর জন্য Circuit Breaker, Retry, Bulkhead, এবং Rate Limiter decorator প্রদানকারী হালকা fault-tolerance library-এর অফিসিয়াল ডকুমেন্টেশন।
- [Netflix Hystrix Wiki (GitHub)](https://github.com/Netflix/Hystrix/wiki) — Netflix-এর মূল circuit breaker library যা microservices জগতে এই প্যাটার্নটিকে জনপ্রিয় করেছিল; প্রকল্পটি এখন maintenance mode-এ, এবং resilience4j-কে এর উত্তরসূরি হিসেবে ব্যাপকভাবে সুপারিশ করা হয়।
- [Microsoft Azure Architecture Center — Circuit Breaker Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker) — state, threshold, এবং implementation বিবেচনা কভার করা canonical প্যাটার্ন রেফারেন্স।
- [Microsoft Azure Architecture Center — Bulkhead Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead) — resource-কে স্বাধীন pool-এ আলাদা করার জন্য canonical প্যাটার্ন রেফারেন্স।
- [Istio — Circuit Breaking](https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/) — service mesh স্তরে circuit breaking-এর জন্য outlier detection এবং connection pool limit কীভাবে configure করবেন।

## পেপারসমূহ

- [Google SRE Book — Chapter 22: Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/) — cascading failure কীভাবে ঘটে এবং সেগুলোর বিরুদ্ধে প্রতিরক্ষা কীভাবে ডিজাইন করতে হয় তার মৌলিক আলোচনা।

## আরও পড়ার জন্য

- [AWS Builders' Library — Timeouts, Retries, and Backoff with Jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) — কেন naive exponential backoff এখনও thundering herd ঘটায়, এবং jitter কীভাবে এটি ঠিক করে তার সুপরিচিত গভীর আলোচনা।
