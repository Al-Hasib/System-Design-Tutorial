# Practice & Interview Questions

**1. Why can't an ordinary in-process mutex coordinate access across multiple servers?**
A mutex relies on the operating system enforcing exclusion over shared memory within a single process. Multiple servers don't share memory, so there's nothing for a mutex to actually synchronize on — the lock state has to live somewhere externally visible to every process, accessed over an unreliable network.

**2. Explain how Redlock decides a lock has been successfully acquired.**
It attempts to acquire the same lock (a key with a unique value and expiry) against multiple independent Redis instances (commonly 5) and considers the lock granted only if a majority (e.g., 3 of 5) succeed within a bounded time window — borrowing the majority-quorum idea from consensus algorithms.

**3. What is the core criticism Martin Kleppmann raises against Redlock?**
That Redlock's safety depends on timing assumptions — bounded clock drift across the Redis instances and no long process pauses (e.g., garbage collection) at the wrong moment — that aren't guaranteed to hold in real systems, meaning two clients could, in rare cases, both believe they hold the same lock.

**4. Why do ZooKeeper and etcd offer stronger correctness guarantees for locking than a single Redis instance or even Redlock?**
They're built directly on a consensus protocol (ZAB for ZooKeeper, Raft for etcd), which maintains one strongly-consistent, agreed-upon view of state across a cluster and tolerates a minority of node failures without ever disagreeing about who holds a lock — rather than relying on independent instances and timing assumptions.

**5. How does a ZooKeeper-based lock detect that a lock holder has crashed, without relying on a timeout guess?**
The lock holder's node is created as "ephemeral," tied to its client session. If the client's session dies (detected via missed heartbeats to the ZooKeeper cluster), the ephemeral node is automatically removed, immediately signaling to the next-in-line contender that the lock is available — without needing to guess an appropriate timeout duration.

**6. Describe the scenario where "holding a lock" doesn't guarantee the protected action happens safely.**
A client acquires a lock, then experiences a long pause (GC, network delay, being descheduled) after acquiring it but before acting on it. Meanwhile, the lock expires and a second client acquires it, does its work, and releases it. When the first client resumes, it may act as if it still safely holds the lock, unaware a second client has already acted — the lock check succeeded, but the assumption that holding it meant exclusive time to act was violated.

**7. What is a fencing token, and how does it solve the problem in question 6?**
A fencing token is a strictly increasing number issued alongside every lock grant. The protected resource (e.g., a storage system) tracks the highest token it has already accepted and rejects any action carrying an older token — so even a delayed client resuming after its lock has effectively expired gets its stale action rejected at the point of actual effect, not just at lock-acquisition time.

**8. Scenario: You need to prevent duplicate execution of a low-stakes scheduled job (e.g., refreshing a cache) that runs on multiple worker instances. Which locking approach would you reach for, and why?**
Redlock against Redis — it's fast, low-operational-overhead, and "good enough" here: an occasional rare duplicate execution of a low-stakes job is an acceptable trade-off for the simplicity of reusing infrastructure you likely already run, rather than standing up a dedicated ZooKeeper/etcd cluster for this.

**9. Scenario: You need to guarantee a financial transaction (e.g., processing a subscription charge) executes exactly once, even under process pauses or network issues. What would you add beyond just acquiring a lock?**
A consensus-backed lock (ZooKeeper or etcd) for stronger correctness than Redlock alone, plus a fencing token checked at the actual point of charging the customer's card — so that even a delayed/"zombie" holder of an expired lock can't cause a real duplicate charge, since the payment system itself would reject a stale token.

**10. True or False: Once a distributed lock is correctly acquired, the action it protects is guaranteed to execute safely with no further precautions needed.**
False. Acquiring the lock only proves the client held exclusive access at the moment of acquisition — a subsequent pause can let that exclusivity lapse before the action actually runs. Fencing tokens (or equivalent mechanisms) are needed at the point of action to guarantee safety, not just at the point of lock acquisition.
