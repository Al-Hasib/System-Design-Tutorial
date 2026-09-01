# Practice & Interview Questions

**1. What is the difference between strong consistency and eventual consistency?**
Strong consistency (linearizability) guarantees that every read sees the most recent write and all clients observe operations in the same real-time order, as if there were a single copy of the data. Eventual consistency only guarantees that replicas converge to the same value if writes stop — it gives no bound on how long convergence takes or what intermediate order clients see. Strong consistency costs coordination and availability; eventual consistency maximizes availability and latency at the cost of possibly stale reads.

**2. A user posts a comment and immediately refreshes but doesn't see it. Which consistency guarantee is missing?**
Read-your-writes consistency. The user's own write should always be visible to their own subsequent reads, even under an otherwise eventually consistent system. This is typically fixed by routing a client's reads to the replica that handled their last write, or by tracking a version/timestamp the client has already seen.

**3. Explain causal consistency and give an example of why it matters.**
Causal consistency guarantees that if operation B depends on operation A (A "happens-before" B), every node sees A before B; unrelated operations may be seen in different orders on different replicas. Example: a comment reply must never appear before the comment it replies to, even though the comment and an unrelated post elsewhere in the feed could be reordered without confusing anyone.

**4. Design an idempotent payment processing API. What would you use as the idempotency key and how would you store it?**
The client generates a unique key (e.g., a UUID) per logical payment attempt — not per network retry — and sends it as a header like `Idempotency-Key`. The server stores `{key -> result}` in a fast, durable store (Redis with TTL, or a database table with a unique constraint on the key) atomically with the actual charge, using a transaction or conditional write. On a retry with the same key, the server looks up the stored result and returns it directly without calling the payment processor again.

**5. Why is idempotency important for the retry pattern discussed earlier in this module?**
Retries are unsafe by default because a client can't distinguish "the request never reached the server" from "the request succeeded but the response was lost." If the underlying operation isn't idempotent, a retry can duplicate effects (e.g., double-charging, duplicate order creation). Idempotency (often via idempotency keys) makes retries safe regardless of which failure mode actually occurred.

**6. What's the difference between PUT and POST in terms of idempotency, and why does that matter for API design?**
PUT is defined by HTTP semantics as idempotent — repeating the same PUT should leave the resource in the same state. POST is not idempotent — repeating it typically creates a new resource each time. This matters because clients and infrastructure (proxies, load balancers) may automatically retry idempotent methods on transient failures, but must not automatically retry POST without an idempotency key, or they risk duplicating side effects.

**7. Why might a system choose causal consistency over strong or eventual consistency?**
Causal consistency captures the ordering guarantees users intuitively expect (e.g., replies after comments, likes after posts) without paying the full coordination cost of linearizability. It's a practical middle ground: more available and lower latency than strong consistency, while avoiding the most confusing artifacts of pure eventual consistency where causally related events can appear out of order.

**8. How do vector clocks help implement causal consistency or detect concurrent writes?**
A vector clock is a per-replica counter vector attached to each write, incremented on each local update and merged on receiving updates from other replicas. By comparing vector clocks, a system can determine whether one write happened-before another (all counters ≤) or whether they were concurrent (neither dominates), which lets it either preserve causal order or flag a conflict for resolution (e.g., last-write-wins or application-level merge).

**9. What tunable consistency options does DynamoDB offer, and what's the trade-off?**
DynamoDB lets you choose "eventually consistent reads" (default, cheaper, lower latency, may return slightly stale data — typically within a second) or "strongly consistent reads" (higher latency and cost, guaranteed to reflect all prior successful writes, and unavailable during certain network issues). The choice is made per read request, letting applications pick the right trade-off per use case.

**10. Why is idempotency not the same thing as consistency, even though they're often discussed together?**
Consistency models describe what data a read is allowed to observe relative to other operations. Idempotency describes whether repeating an operation changes the outcome. A system can be strongly consistent yet have non-idempotent operations (e.g., a linearizable "increment counter" operation), and a system can be eventually consistent while every operation is idempotent (e.g., idempotent "set" operations that just need last-write-wins). They interact — retries under weak consistency need extra care — but they answer different questions.

**11. In a system using session consistency (read-your-writes + monotonic reads), what could two different users legitimately see at the same moment?**
Two different users could see different snapshots of global state — user A might see a comment that user B doesn't yet see, because session consistency only guarantees a single client's own view is coherent over time, not that all clients converge instantly. This is acceptable for many consumer apps because each user only notices inconsistencies in their own interaction history, not in comparison to other users' views.

**12. How would you retrofit idempotency onto an existing non-idempotent "POST /transfer" endpoint that moves money between accounts?**
Add a required `Idempotency-Key` (or `transfer_id`) parameter generated client-side per transfer attempt. Add a unique constraint on that key in the transfers table, and process the transfer plus the key-insertion in a single atomic transaction. On any retry with the same key, the insert fails the uniqueness check (or a lookup finds the existing row), and the server returns the previously recorded result instead of moving money again.
