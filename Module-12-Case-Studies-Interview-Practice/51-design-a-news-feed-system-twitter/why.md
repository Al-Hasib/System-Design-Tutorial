# Why This Topic Matters: Design a News Feed System (like Twitter/Facebook)

> **In one sentence:** This is the problem where one decision — compute the feed when someone writes, or when someone reads — determines the entire architecture, and where the correct answer is "both, depending on the user," which is why it is the best trade-off question in system design.

## Why This Case Study Exists

A news feed looks like a simple query: get recent posts from the people I follow, sorted by time. On a small dataset it is exactly that query.

At scale it becomes impossible. A user following 2,000 accounts triggers a query across 2,000 partitions, sorted and merged, on every feed refresh, for hundreds of millions of users. The obvious fix — precompute everyone's feed when a post is created — works beautifully until someone with 100 million followers posts, and one write turns into 100 million writes.

Both extremes fail. The design is the negotiation between them, and that is the whole point of the exercise.

## The Design Problems It Forces You to Solve

### 1. Fan-out on write versus fan-out on read
**The problem:** Feed assembly is expensive, and you must pay for it somewhere.

**Why it is hard:** The two options have opposite cost profiles and opposite failure modes.

**What you learn:** **Fan-out on write** (push): when a user posts, append the post ID to every follower's precomputed feed list. Reads become a single fast lookup — perfect for a read-heavy product — but writes are amplified by the follower count, and a celebrity post becomes a write storm. **Fan-out on read** (pull): store posts once and assemble the feed at request time. Writes are cheap and constant; reads are expensive and slow, especially for users following many accounts.

**The synthesis that makes this problem famous:** use fan-out on write for ordinary users, and fan-out on read for the small number of accounts with enormous follower counts. A user's feed is then the merge of their precomputed list plus a live query for the handful of celebrities they follow. Arriving at that hybrid — and explaining *why* neither pure approach works — is the single highest-value answer in this interview.

### 2. Storage that grows faster than the content
**The problem:** Precomputed feeds duplicate post references across every follower, so storage scales with the *follow graph*, not with the number of posts.

**What you learn:** The mitigations, each with a cost. Store post IDs rather than post content and hydrate at read time (less storage, an extra lookup). Cap each feed at a few hundred entries, since almost nobody scrolls past that. Skip fan-out entirely for inactive users and compute their feed on demand when they return — a genuinely large saving, because the long tail of dormant accounts is most of the user base.

### 3. Feed generation that cannot block the post request
**The problem:** A user posts and must get an immediate confirmation, but fanning out to a million followers takes time.

**What you learn:** The write path persists the post and publishes an event; workers consume it and do the fan-out asynchronously. This makes the feed **eventually consistent** — a follower may not see the post for a few seconds — and teaches you to check whether that is acceptable (for a social feed, obviously yes; for a stock price, obviously no). It also introduces the operational reality of monitoring consumer lag, since a backed-up fan-out queue is how feeds silently go stale.

### 4. Ranking, and why it changes the data model
**The problem:** A chronological feed is a sorted merge. A ranked feed ("top posts for you") requires scoring candidates by engagement, recency, affinity, and a model.

**What you learn:** That ranking splits the read path into candidate generation and scoring, adds a feature store and a model-serving dependency, and makes latency budgets much tighter. It also makes pagination harder: with a shifting ranked list, offset-based paging shows duplicates and skips items, which is why **cursor-based pagination** is the correct choice — a small, specific detail that interviewers notice.

### 5. Read volume that dwarfs everything else
**The problem:** Feed reads are one of the highest-volume operations in any consumer product.

**What you learn:** Caching is not an optimization here, it is the architecture. Precomputed feeds live in Redis; post content is cached separately and hydrated; media is served entirely from a CDN. And you learn to ask the estimation question that drives all of it — daily active users times refreshes per day divided by seconds — because that number decides how many cache nodes appear in your diagram.

## What It Costs to Get Wrong

- **Committing to one fan-out strategy** without discussing the other is the most common weakness in this interview, and it is the exact thing being tested.
- **Ignoring the celebrity problem** produces a design that works for 99.99% of users and falls over for the accounts that generate the most traffic.
- **Fanning out synchronously** on the post request couples posting latency to follower count.
- **Offset pagination on a ranked feed** produces duplicate and missing items, a bug users see immediately.
- **Forgetting storage amplification** understates infrastructure cost by an order of magnitude.

## Why Interviewers Choose This One

There is no single right answer, which makes it an excellent instrument for assessing judgment rather than recall. The interviewer can push in either direction — "what if we have a user with 200 million followers?", "what if storage cost is the constraint?", "what if the feed must be ranked?" — and watch the candidate adapt. It also exercises nearly every earlier topic at once: caching, sharding, queues, denormalization, eventual consistency, and CDN offload.

## How It Connects

This case study applies **caching** at load-bearing scale (topic 17), **Redis** data structures for feed lists (topic 19), **message queues** and asynchronous **fan-out** (topics 20, 21), **denormalization** as a deliberate read optimization (topic 16), **eventual consistency** (topic 29), **sharding** of the post and graph stores (topic 14), **CDN** delivery for media (topic 18), **Bloom filters** for "has this user already seen this post?" (topic 42), and **batch versus stream processing** for trending and ranking signals (topic 23).

**Next:** [Design a Distributed File Storage System](../52-design-a-distributed-file-storage-google-drive/why.md) — where the hard parts move from fan-out to chunking, deduplication, and sync conflicts.
