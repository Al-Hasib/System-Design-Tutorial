# আরও পড়াশোনা ও রেফারেন্স

## Official ডকুমেন্টেশন

- [X/Open XA Specification (The Open Group-এর মাধ্যমে সংক্ষিপ্ত বিবরণ)](https://pubs.opengroup.org/onlinepubs/009680699/toc.pdf) — বেশিরভাগ বাস্তব-জগতের 2PC/XA transaction manager implementation-এর অন্তর্নিহিত standard specification।
- [Temporal Documentation: Saga Pattern](https://docs.temporal.io/) — saga এবং compensating workflow implement করার জন্য first-class সমর্থন-সহ durable execution engine।
- [AWS Step Functions Documentation](https://docs.aws.amazon.com/step-functions/) — serverless/cloud architecture-এ orchestration-based saga implementation-এর জন্য সাধারণত ব্যবহৃত।

## গবেষণাপত্র

- Gray, J. & Lamport, L., ["Consensus on Transaction Commit"](https://www.microsoft.com/en-us/research/publication/consensus-on-transaction-commit/) — atomic commit protocol (2PC/3PC)-কে consensus-এর সাথে সংযুক্তকারী Microsoft Research গবেষণাপত্র।
- Garcia-Molina, H. & Salem, K., "Sagas" (ACM SIGMOD, 1987) — compensating transaction-সহ transaction-এর একটি sequence হিসেবে Saga ধারণা প্রবর্তনকারী মূল গবেষণাপত্র।

## আরও পড়াশোনা

- [microservices.io: Pattern — Saga](https://microservices.io/patterns/data/saga.html) — choreography বনাম orchestration saga-র Chris Richardson-এর ব্যাপকভাবে উদ্ধৃত বিশ্লেষণ।
- [Wikipedia: Two-Phase Commit Protocol](https://en.wikipedia.org/wiki/Two-phase_commit_protocol) — protocol, এর phase, এবং পরিচিত failure mode-এর একটি সুদৃঢ় সংক্ষিপ্ত বিবরণ।
- [Wikipedia: Compensating Transaction](https://en.wikipedia.org/wiki/Compensating_transaction) — দীর্ঘস্থায়ী business process-এ ব্যবহৃত compensating transaction-এর পটভূমি।
