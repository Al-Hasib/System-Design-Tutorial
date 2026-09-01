# Practice & Interview Questions

**1. Why can't wall-clock timestamps from different machines be trusted to order events across a distributed system?**
Every machine has its own physical clock, and even with NTP synchronization, clocks drift and skew relative to each other — sometimes by milliseconds, sometimes more under network issues. If two events on different machines happen close together, comparing their wall-clock timestamps can give a wrong answer about which actually happened first, with no way to detect the error after the fact.

**2. Describe the two rules that define a Lamport timestamp.**
Every local event increments the process's own counter. Whenever a process sends a message, it includes its current counter value; the receiving process sets its own counter to the maximum of its own current value and the received value, then increments it by one.

**3. What guarantee does a Lamport timestamp give you, and what can't it tell you?**
If event A happened-before event B (A could have causally influenced B), then A's Lamport timestamp is guaranteed to be smaller than B's. It cannot tell you the reverse — two events with no causal relationship (truly concurrent, unrelated events) might get timestamps in either order or coincidentally similar values, so you can't use Lamport timestamps alone to detect true concurrency.

**4. How does a vector clock differ structurally from a Lamport timestamp?**
A Lamport timestamp is a single counter. A vector clock is an array of counters, one slot per process in the system — each process increments only its own slot on a local event, and takes the element-wise maximum with a received vector (then increments its own slot) when receiving a message.

**5. How do you determine, using two vector clocks, whether two events are truly concurrent?**
Compare them element-wise. If every element of vector A is less than or equal to the corresponding element of vector B (with at least one strictly less), A happened-before B. If neither vector dominates the other — some elements are higher in A, others higher in B — the events are truly concurrent, meaning neither could have causally influenced the other.

**6. Why does Amazon's Dynamo use vector clocks instead of simple "last write wins" timestamp comparison?**
Because "last write wins" based on wall-clock timestamps can silently and incorrectly discard a legitimate concurrent update due to clock skew. Vector clocks let Dynamo correctly detect when two writes are genuinely concurrent and conflicting, so the conflict can be explicitly merged or surfaced for resolution rather than one write being silently and possibly wrongly overwritten.

**7. Give two examples of when physical (NTP-synchronized) wall-clock time is the right tool, despite its limitations for ordering.**
Any two of: timestamps in logs for human debugging, cache TTL/expiration windows, or rate-limiting time windows — cases where an approximate, human-meaningful sense of "when" is sufficient and correctness doesn't depend on precise causal ordering between machines.

**8. Scenario: A distributed cache's cluster nodes need to agree on a per-key expiration time accurate to within a few seconds. Would you use physical time or a logical/vector clock here, and why?**
Physical (NTP-synchronized) time — TTL expiration only needs an approximate, human-meaningful sense of "when," and a few seconds of clock skew is an acceptable margin of error for this use case; a logical/vector clock adds complexity this scenario doesn't need.

**9. Scenario: Two replicas of a distributed database each receive an update to the same record while partitioned from each other, and you need to correctly detect this as a conflict rather than silently picking one via timestamp. Which tool from this video applies, and why?**
Vector clocks — comparing the two updates' vector clocks would show that neither dominates the other, correctly identifying them as concurrent, conflicting writes that need explicit merge/resolution logic, rather than trusting wall-clock timestamps that clock skew could make unreliable.

**10. True or False: A vector clock can always tell you which of two events happened "first" in real, physical time.**
False. A vector clock tells you about causal ordering (happened-before) and can detect concurrency, but it says nothing about physical/wall-clock time — two concurrent events (by vector clock) may have occurred at genuinely different real-world instants; the point of a vector clock is that "which one happened first in physical time" is often not even the meaningful or answerable question for correctness purposes.
