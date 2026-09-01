# Further Reading & References

## Official Docs

- [The Raft Consensus Algorithm (raft.github.io)](https://raft.github.io/) — the official Raft website, including the well-known interactive visualization of leader election and log replication.
- [etcd Documentation — Raft in etcd](https://etcd.io/docs/latest/learning/) — how etcd implements the Raft consensus protocol as its replication layer, used as the Kubernetes control-plane store.

## Papers

- Leslie Lamport, ["The Part-Time Parliament"](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf) (ACM Transactions on Computer Systems, 1998) — the original Paxos paper, framed as an allegory about a legislature on the fictional Greek island of Paxos.
- Leslie Lamport, ["Paxos Made Simple"](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) (ACM SIGACT News, 2001) — Lamport's own, more direct re-explanation of Paxos, written because the original paper was too hard to follow.
- Diego Ongaro and John Ousterhout, ["In Search of an Understandable Consensus Algorithm (Raft)"](https://raft.github.io/raft.pdf) (USENIX ATC, 2014) — the original Raft paper, explicitly designed around understandability as a first-class goal.

## Further Reading

- [Wikipedia — Paxos (computer science)](https://en.wikipedia.org/wiki/Paxos_(computer_science)) — accessible overview of Paxos roles, phases, and variants (Multi-Paxos, Fast Paxos).
- [Wikipedia — Raft (algorithm)](https://en.wikipedia.org/wiki/Raft_(algorithm)) — accessible overview of Raft's leader election, log replication, and safety properties.
