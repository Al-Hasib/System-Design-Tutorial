# Follow-Up Interview Questions

Use these to test depth after the main design. Model answers are concise — expand verbally in a real interview.

**1. How would you handle a celebrity account with 50 million followers?**
Skip fan-out-on-write for that account. Detect it by follower count crossing a threshold, and instead serve their posts via fan-out-on-read: followers merge the celebrity's recent posts into their own precomputed timeline at read time. This avoids a 50-million-write burst on every post and prevents the hot-key/hot-shard problem in the cache layer.

**2. How would you rank feed items instead of showing them in pure chronological order?**
Score each candidate post with a model using signals like recency, author affinity (how often the user engages with this author), predicted engagement (likes/comments/shares probability), and content type. Precompute a rough score at fan-out time so it can be inserted into the sorted-set cache by score, then optionally re-rank the top N candidates at read time with fresher signals for latency-sensitive personalization.

**3. How would you handle a user who follows 10,000 accounts?**
Even with fan-out-on-write for most of those accounts, reading and merging thousands of contributions is manageable because writes already pre-populated their cached timeline — the read is still a single cache fetch. The heavier cost is on the write side: every one of those 10,000 followees fans out to this user, so the user's timeline update rate is high; cap the cached timeline length (e.g., latest ~800 entries) and rely on eviction plus rebuild-on-miss to bound memory.

**4. How would you backfill a brand-new user's feed when they have no cached timeline yet?**
On first login (cache miss), fall back to a fan-out-on-read path: query the follow graph for who they follow, pull each followee's recent posts from the sharded post store, merge and rank them, return the result, and asynchronously populate the cache so subsequent reads are fast.

**5. How do you propagate a deleted or edited post to feeds it has already been fanned out to?**
Store posts by ID and fetch full content at read time (hydration) rather than duplicating content into every timeline entry — the cache stores post IDs, not full post bodies. On delete, mark the post as deleted (or remove it) in the sharded post store and let the Feed Service filter out or refresh missing/deleted IDs during hydration. On edit, the post store is the source of truth, so cached ID references automatically show updated content — no fan-out re-push needed.

**6. Should ranking updates be computed in real time (stream) or in batch?**
Use a hybrid, referencing the batch vs. stream trade-off: lightweight, latency-sensitive signals (recency, immediate engagement counts) are updated via a stream processing pipeline for freshness; heavier signals (longer-term affinity models, engagement-prediction model weights) are retrained and refreshed via periodic batch jobs since they change more slowly and are more compute-intensive.

**7. How do you keep the follow-graph store performant when celebrity accounts have tens of millions of followers?**
Shard the follow-graph store separately from the post store, since access patterns differ (graph traversal vs. key-value post lookup). For celebrity accounts, avoid ever loading their full follower list into a single fan-out job; instead paginate/stream it in batches through the queue, or skip loading it entirely by using fan-out-on-read for those accounts.

**8. What happens if the Fan-out Service falls behind or crashes mid-fan-out?**
Because the Post Service publishes to a durable message queue (Kafka) rather than fanning out synchronously, the post itself is safely persisted regardless of fan-out progress. The Fan-out Service can resume from its last committed offset, and consumers can be scaled horizontally to catch up. Followers may briefly see a delayed post — acceptable under our eventual consistency requirement.

**9. How would you avoid a single Redis node becoming a bottleneck for a trending celebrity post?**
Use consistent hashing across the Redis cluster to spread keys evenly, then add targeted mitigations for the specific hot key: read replicas for that shard, short-TTL local/edge caching in front of it, or request coalescing so many simultaneous reads for the same key collapse into one backend fetch.

**10. How do you test that the ranking algorithm change didn't regress engagement?**
Roll out via A/B testing: route a percentage of users to the new ranking model, compare engagement metrics (time spent, likes, shares, session return rate) against the control group, and only ramp up if the treatment group shows statistically significant improvement without harming other metrics like content diversity.

**11. How would you support real-time notifications (e.g., "X liked your post") without duplicating the whole feed pipeline?**
Reuse the existing pub-sub backbone: emit a lightweight event on the same or a parallel Kafka topic for the interaction, and have a separate Notification Service subscribe to it — decoupled from the Fan-out Service, since notifications have different delivery guarantees (usually at-least-once, low latency) than the feed's precomputed timelines.

**12. How would you scale the post store and cache as the user base doubles?**
Add more shards/nodes to both the sharded post store and the Redis cache cluster; because both use consistent hashing, only a fraction of keys need to move to the new nodes rather than a full remap, minimizing cache-miss storms and rebalancing cost during the scale-out.
