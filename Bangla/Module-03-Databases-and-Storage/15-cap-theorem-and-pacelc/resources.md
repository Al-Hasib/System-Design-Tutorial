# আরও পড়াশোনা ও রেফারেন্স

## Papers

- Eric Brewer, ["CAP Twelve Years Later: How the 'Rules' Have Changed"](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/) — Brewer-এর নিজের 2012 সালের retrospective, যা মূল conjecture-টি পুনর্বিবেচনা ও স্পষ্ট করে।
- Seth Gilbert ও Nancy Lynch, ["Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services"](https://groups.csail.mit.edu/tds/papers/Gilbert/Brewer2.pdf) — CAP theorem-এর আনুষ্ঠানিক প্রমাণ, Brewer-এর মূল PODC 2000 keynote conjecture-এর উপর ভিত্তি করে।
- Daniel J. Abadi, ["Consistency Tradeoffs in Modern Distributed Database System Design"](https://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf) — CAP-এর একটি extension হিসেবে PACELC framework প্রবর্তনকারী paper।

## অফিসিয়াল ডকুমেন্টেশন

- [Amazon DynamoDB: Read Consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html) — DynamoDB-তে eventually consistent বনাম strongly consistent read।
- [Apache Cassandra Documentation: Consistency](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html#tunable-consistency) — Cassandra-তে টিউনযোগ্য consistency level (ONE, QUORUM, ALL)।
- [MongoDB Documentation: Read Concern](https://www.mongodb.com/docs/manual/reference/read-concern/) এবং [Write Concern](https://www.mongodb.com/docs/manual/reference/write-concern/) — MongoDB প্রতি operation-এ consistency বনাম availability/latency কীভাবে টিউন করে।

## আরও পড়াশোনা

- [Wikipedia: CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem) — theorem-টির সহজবোধ্য পর্যালোচনা, এর ইতিহাস, এবং সাধারণ সমালোচনা।
- [Wikipedia: PACELC theorem](https://en.wikipedia.org/wiki/PACELC_design_principle) — PACELC extension-এর পর্যালোচনা।
- [Wikipedia: Eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency) — বেশিরভাগ AP system reconciliation-এর জন্য যে consistency model-এর উপর নির্ভর করে, তার পটভূমি।
