# Further Reading & References

## Official Docs

- [Amazon DynamoDB — Read Consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html) — official documentation on eventually consistent vs strongly consistent reads.
- [Stripe API — Idempotent Requests](https://stripe.com/docs/api/idempotent_requests) — official documentation on using idempotency keys to safely retry API requests.

## Papers

- Werner Vogels, ["Eventually Consistent"](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html) — widely cited explanation of eventual consistency from Amazon's CTO, originally published in ACM Queue.

## Further Reading

- Wikipedia, ["Consistency model"](https://en.wikipedia.org/wiki/Consistency_model) — overview of the full spectrum of consistency models including linearizability, sequential, and causal consistency.
- Wikipedia, ["Eventual consistency"](https://en.wikipedia.org/wiki/Eventual_consistency) — background and formal definition.
- Wikipedia, ["Linearizability"](https://en.wikipedia.org/wiki/Linearizability) — formal definition and history of the strong consistency model.
- [Jepsen.io](https://jepsen.io/) — independent analyses of real distributed databases' actual consistency guarantees under network partitions and faults.
