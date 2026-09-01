# Further Reading & References

## Papers

- Eric Brewer, ["CAP Twelve Years Later: How the 'Rules' Have Changed"](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/) — Brewer's own 2012 retrospective revisiting and clarifying the original conjecture.
- Seth Gilbert & Nancy Lynch, ["Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services"](https://groups.csail.mit.edu/tds/papers/Gilbert/Brewer2.pdf) — the formal proof of the CAP theorem, based on Brewer's original PODC 2000 keynote conjecture.
- Daniel J. Abadi, ["Consistency Tradeoffs in Modern Distributed Database System Design"](https://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf) — the paper introducing the PACELC framework as an extension of CAP.

## Official Docs

- [Amazon DynamoDB: Read Consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html) — eventually consistent vs. strongly consistent reads in DynamoDB.
- [Apache Cassandra Documentation: Consistency](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html#tunable-consistency) — tunable consistency levels (ONE, QUORUM, ALL) in Cassandra.
- [MongoDB Documentation: Read Concern](https://www.mongodb.com/docs/manual/reference/read-concern/) and [Write Concern](https://www.mongodb.com/docs/manual/reference/write-concern/) — how MongoDB tunes consistency vs. availability/latency per operation.

## Further Reading

- [Wikipedia: CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem) — accessible overview of the theorem, its history, and common criticisms.
- [Wikipedia: PACELC theorem](https://en.wikipedia.org/wiki/PACELC_design_principle) — overview of the PACELC extension.
- [Wikipedia: Eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency) — background on the consistency model most AP systems rely on for reconciliation.
