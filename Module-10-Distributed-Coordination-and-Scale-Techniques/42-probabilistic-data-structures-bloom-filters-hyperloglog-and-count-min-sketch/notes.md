# Study Notes: Probabilistic Data Structures

## Definitions

- **Probabilistic data structure:** A structure that answers a query approximately, using a fixed, small amount of memory, in exchange for a controlled, bounded error rate.
- **Bloom filter:** A bit array + multiple hash functions answering "is this element possibly in the set?" — no false negatives, tunable false-positive rate.
- **False positive:** Filter says "present" but the element was never actually added (possible with Bloom filters, due to hash collisions across elements).
- **False negative:** Filter says "not present" but the element was actually added (impossible with Bloom filters, by construction).
- **HyperLogLog:** A structure estimating the cardinality (count of unique items) of a massive stream using kilobytes of memory, based on statistics of hash-value leading zeros.
- **Count-Min Sketch:** A 2D array of counters + multiple hash functions estimating item frequency in a stream, with one-directional (overestimate-only) error.

## Bloom Filter Mechanics

| Operation | How it works |
|---|---|
| Add element | Hash with k independent hash functions; set each resulting bit position to 1 |
| Check membership | Hash the same way; if ANY bit is 0 → definitely not present. If ALL bits are 1 → probably present (may be a false positive) |
| Tuning | Larger bit array + more hash functions = lower false-positive rate, at the cost of more memory |

## The Three Structures Compared

| Structure | Question answered | Error type | Typical use |
|---|---|---|---|
| Bloom filter | Is this element in the set? | False positives possible; false negatives impossible | Skip expensive lookups (e.g., LSM-tree SSTable checks), dedup at scale, crawler "seen URL?" checks |
| HyperLogLog | How many unique items? | ~1-2% relative error either direction | Unique visitor counts, unique event counts at massive scale (Redis `PFCOUNT`) |
| Count-Min Sketch | How often has this item occurred? | Overestimate only (never underestimates) | Top-K / trending items, approximate frequency counting at scale |

## HyperLogLog Intuition

- A hash value with N leading zero bits is roughly a 1-in-2^N event.
- Observing hash values with long runs of leading zeros is statistical evidence of having hashed many distinct items.
- HyperLogLog buckets incoming items (e.g., into 16,384 buckets) and tracks the max leading-zero count per bucket, then combines all buckets via a specific averaging formula for the final estimate.
- Result: cardinality estimates accurate to ~1-2% using only a few KB of memory, regardless of whether the true count is thousands or billions.

## Count-Min Sketch Mechanics

| Operation | How it works |
|---|---|
| Record occurrence | Hash item with k hash functions (each mapping to a different row); increment the counter at each resulting cell |
| Estimate frequency | Hash the same way; take the MINIMUM of all counters landed on (collisions can only inflate a counter, never deflate it, so the minimum is closest to correct) |

## Key Numbers / Facts

- Bloom filters were introduced by Burton Howard Bloom in 1970.
- HyperLogLog (Flajolet et al., 2007) typically achieves ~1-2% standard error using around 1.5 KB of memory for cardinalities up to billions.
- Redis implements HyperLogLog natively (`PFADD`, `PFCOUNT`, `PFMERGE`).

## Summary

- All three structures trade a small, mathematically-bounded error for dramatic, fixed memory savings compared to exact tracking.
- Bloom filters: membership, no false negatives. HyperLogLog: cardinality, ~1-2% error. Count-Min Sketch: frequency, overestimate-only error.
- Use these when an approximate answer produces the same practical business outcome as an exact one — never for correctness-critical exact values like financial balances.
