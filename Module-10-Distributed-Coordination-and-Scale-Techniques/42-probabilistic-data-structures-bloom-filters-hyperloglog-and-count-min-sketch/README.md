# Probabilistic Data Structures: Bloom Filters, HyperLogLog & Count-Min Sketch

**Difficulty:** Advanced
**Estimated length:** 14-18 min
**Prerequisites:** [24 - Consistent Hashing Explained](../../Module-06-Distributed-Systems-Concepts/24-consistent-hashing-explained/README.md), [19 - Distributed Caching with Redis & Memcached](../../Module-04-Caching-and-Content-Delivery/19-distributed-caching-redis-and-memcached/README.md)

## Learning Objectives

- Explain the core idea behind probabilistic data structures: trading a small, bounded error rate for massive memory savings.
- Describe how a Bloom filter answers "have I seen this before?" and why it can false-positive but never false-negative.
- Explain how HyperLogLog estimates the number of unique items in a massive stream using a tiny, fixed amount of memory.
- Explain how a Count-Min Sketch estimates the frequency of items in a stream with bounded, one-directional error.
- Recognize system design scenarios where an approximate answer is not just acceptable but the only practical option.

## Script

### Hook / Intro

Some questions in system design have an obvious, exact answer that's simply too expensive to compute at scale. "Have we already processed this exact request ID, out of the billion we've seen this month?" "How many unique visitors did our site have today, out of hundreds of millions of page views?" "Roughly how often is this specific URL being requested, out of a firehose of millions of requests per second?" Storing every single ID, every unique visitor, every URL's exact count — that's a real, and sometimes prohibitive, amount of memory. Today we cover the three probabilistic data structures that show up constantly once "exact but expensive" stops being an acceptable answer: Bloom filters, HyperLogLog, and Count-Min Sketch. Each one deliberately trades a small, precisely-bounded error for a dramatic reduction in memory.

### Bloom Filters: "Have I Seen This Before?"

A Bloom filter answers set-membership questions — "is this element possibly in the set?" — using a fixed-size bit array and several independent hash functions, instead of storing the actual elements. To add an element, you run it through each hash function, and set the bit at each resulting position in the array to 1. To check whether an element might be in the set, you hash it the same way and check whether *all* those bit positions are set to 1. If even one of them is 0, the element is definitely not in the set. If all of them are 1, the element is *probably* in the set — but this could be a **false positive**, where unrelated elements happened to collectively flip on the exact same bits. Critically, a Bloom filter can never produce a **false negative** — if it says "not present," that's always correct, because adding an element only ever sets bits, never unsets them.

This asymmetry is exactly why Bloom filters are so useful as a cheap, fast pre-check in front of an expensive, authoritative check: recall from the LSM-tree video that a database read can skip an entire SSTable if its Bloom filter says the key definitely isn't there — saving a disk read for the common case, and only paying for a slower, exact check on the rare false positive. Other classic uses: a web crawler checking "have I already crawled this URL?" without storing every URL ever seen, or a spam filter checking "have I seen this exact hash of malicious content before?" The size of the bit array and the number of hash functions tune the false-positive rate — bigger array and more hash functions mean fewer false positives, at the cost of more memory.

### HyperLogLog: "Roughly How Many Unique Items?"

Counting exact unique items (cardinality) over a huge stream naively requires storing every unique item you've seen — a hash set, essentially — which can require enormous memory for billions of items. **HyperLogLog** estimates this count using a clever observation about hash function outputs: for a random-looking hash, the probability of seeing a hash value with, say, 10 leading zero bits is 1 in 1,024 — so if you've observed a hash with that many leading zeros, it's statistical evidence that you've probably hashed roughly that many *distinct* items (since more distinct items means more chances to see a hash with an unusually long run of leading zeros). HyperLogLog tracks the maximum number of leading zeros observed across many independent buckets (hashing each incoming item into one of, say, 16,384 buckets, and tracking the max leading-zero count per bucket), then combines all the buckets' estimates using a specific averaging formula to get one final cardinality estimate. The result is genuinely remarkable: HyperLogLog can estimate the cardinality of a set with billions of unique elements to within roughly 1-2% error, using only a few kilobytes of memory — regardless of whether the true cardinality is a thousand or a billion. This is exactly what powers "unique visitors" counters in analytics systems like Redis's `PFCOUNT` and most large-scale web analytics platforms.

### Count-Min Sketch: "Roughly How Often?"

Where Bloom filters answer "is it in the set" and HyperLogLog answers "how many unique items," a **Count-Min Sketch** answers "approximately how many times has this specific item occurred?" — a frequency-estimation problem, using a 2D array of counters and multiple hash functions. To record an occurrence of an item, you hash it with each of several hash functions (each mapping to a different row of the 2D array) and increment the counter at each resulting position. To estimate an item's frequency, you hash it the same way and take the *minimum* of all the counters you land on — taking the minimum specifically because hash collisions can only ever inflate a counter (multiple different items incrementing the same cell), never deflate it, so the smallest of several independent estimates is the closest to correct. Like a Bloom filter, this means Count-Min Sketch's error is strictly one-directional: it can overestimate an item's true frequency (due to collisions) but will never underestimate it. This is the tool behind "trending topics" and "top-K most frequent items" features at massive scale — you genuinely cannot afford an exact counter for every possible search term, hashtag, or URL a system might see, but you can afford a fixed-size sketch that gives you a reliably close estimate for the ones that actually matter.

### The Common Thread

All three structures share the same underlying trade: a fixed, small memory footprint that doesn't grow with the size of the data you're summarizing, in exchange for a controlled, mathematically-understood error rate — never an unbounded or unpredictable one. None of them are appropriate when you need an exact answer (a bank balance, an inventory count that must never be wrong) — they're specifically for the class of problems where "exactly right, at massive memory cost" and "very close, at trivial memory cost" produce the same practical business outcome, which turns out to be a surprisingly large class of real system design problems.

### Real-World Example

Consider a large-scale ad click tracking system. Deduplicating clicks (has this exact click ID already been recorded, out of billions per day?) is a natural Bloom filter application — a cheap first check before touching the authoritative, exact data store, with the rare false positive costing only a slightly-too-cautious skip rather than any real correctness problem. Counting unique users who clicked a given ad campaign uses HyperLogLog — advertisers care about "roughly how many unique people," not an exact-to-the-person count, and this can run over billions of events using kilobytes of memory instead of terabytes. And identifying the top 10 highest-clicking ad campaigns out of millions of active campaigns in near real-time uses a Count-Min Sketch to track approximate click frequency per campaign ID, cheaply enough to update on every single click, surfacing the genuinely high-frequency campaigns without needing an exact counter for every campaign that will never be in the top 10 anyway.

### Recap

Probabilistic data structures trade a small, precisely-bounded error rate for a dramatic reduction in memory, and are the right tool exactly when an approximate answer is genuinely good enough for the business problem. Bloom filters answer set-membership questions with zero false negatives and a tunable false-positive rate — a cheap pre-check in front of an expensive exact lookup. HyperLogLog estimates unique counts (cardinality) over massive streams using kilobytes of memory and roughly 1-2% error. Count-Min Sketch estimates item frequencies with one-directional (overestimate-only) error, powering top-K and trending-item features at scale. None of the three should be reached for when an answer must be exact — they earn their place specifically where "close enough" and "exactly right" produce the same real-world outcome.

### What's Next

That closes out this module on distributed coordination and scale techniques. Next module shifts to a different, equally essential concern: how you actually observe, deploy, and operate a system like this once it's live — logging, metrics, tracing, containers, deployment strategies, and how to test whether a distributed system actually survives failure.

## Key Takeaways

- Probabilistic data structures trade a small, mathematically-bounded error rate for dramatically reduced, fixed memory usage compared to storing exact data.
- A Bloom filter answers "have I seen this?" with zero false negatives (if it says no, that's certain) and a tunable false-positive rate — a cheap pre-check before an expensive, authoritative lookup.
- HyperLogLog estimates the number of unique items (cardinality) in a massive stream to within roughly 1-2% error using only kilobytes of memory, regardless of the true cardinality.
- Count-Min Sketch estimates item frequency with one-directional (overestimate-only) error, powering top-K/trending-item features without needing an exact counter per item.
- None of these are appropriate when an answer must be exact (financial balances, inventory) — they're for the large class of problems where "close enough" and "exactly right" produce the same practical outcome.
