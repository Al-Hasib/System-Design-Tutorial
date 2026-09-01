# Further Reading & References

## Official Docs

- [X/Open XA Specification (overview via The Open Group)](https://pubs.opengroup.org/onlinepubs/009680699/toc.pdf) — the standard specification underlying most real-world 2PC/XA transaction manager implementations.
- [Temporal Documentation: Saga Pattern](https://docs.temporal.io/) — durable execution engine with first-class support for implementing sagas and compensating workflows.
- [AWS Step Functions Documentation](https://docs.aws.amazon.com/step-functions/) — commonly used for orchestration-based saga implementations in serverless/cloud architectures.

## Papers

- Gray, J. & Lamport, L., ["Consensus on Transaction Commit"](https://www.microsoft.com/en-us/research/publication/consensus-on-transaction-commit/) — Microsoft Research paper connecting atomic commit protocols (2PC/3PC) with consensus.
- Garcia-Molina, H. & Salem, K., "Sagas" (ACM SIGMOD, 1987) — the original paper introducing the Saga concept as a sequence of transactions with compensating transactions.

## Further Reading

- [microservices.io: Pattern — Saga](https://microservices.io/patterns/data/saga.html) — Chris Richardson's widely-referenced breakdown of choreography vs orchestration sagas.
- [Wikipedia: Two-Phase Commit Protocol](https://en.wikipedia.org/wiki/Two-phase_commit_protocol) — solid overview of the protocol, its phases, and known failure modes.
- [Wikipedia: Compensating Transaction](https://en.wikipedia.org/wiki/Compensating_transaction) — background on compensating transactions as used in long-running business processes.
