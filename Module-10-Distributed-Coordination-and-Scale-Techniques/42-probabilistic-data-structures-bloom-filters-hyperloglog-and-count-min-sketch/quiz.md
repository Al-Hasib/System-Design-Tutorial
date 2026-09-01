# Practice & Interview Questions

**1. What core trade-off do all probabilistic data structures make?**
They trade a small, mathematically-bounded error rate for a dramatic, fixed reduction in memory usage compared to storing exact data — and that memory footprint typically doesn't grow with the size of the underlying dataset the way exact storage would.

**2. Why can a Bloom filter produce false positives but never false negatives?**
Adding an element only ever sets bits in the array to 1, never unsets them. If any bit checked for a given element is still 0, that element was definitely never added — a certain answer. But if all checked bits are 1, it might be because other, unrelated elements collectively happened to set exactly those same bits (a false positive), even though this specific element was never added.

**3. Why is a Bloom filter useful as a pre-check in front of an expensive, authoritative lookup (e.g., an LSM-tree SSTable check)?**
Because a "definitely not present" answer from the Bloom filter lets you skip the expensive lookup entirely with full confidence, and this is the common case for a well-tuned filter. The rare false positive only costs an unnecessary but harmless expensive lookup — never a wrong final answer, since the authoritative check still happens whenever the filter says "maybe."

**4. Explain the statistical idea behind HyperLogLog's cardinality estimation.**
For random-looking hash values, observing an unusually long run of leading zero bits is rare — a hash with N leading zeros has roughly a 1-in-2^N chance. Seeing such long runs at all is therefore statistical evidence that many distinct items have been hashed, since more distinct items means more chances to observe a rare, long run. HyperLogLog tracks the maximum leading-zero run per bucket across many buckets and combines them into one cardinality estimate.

**5. Roughly what accuracy and memory footprint does HyperLogLog achieve, regardless of the true cardinality?**
Roughly 1-2% relative error, using only a few kilobytes of memory — whether the true number of unique items is in the thousands or in the billions, the memory usage stays essentially fixed.

**6. How does a Count-Min Sketch estimate an item's frequency, and why does it take the minimum across hashed positions rather than, say, the average?**
It hashes the item with several independent hash functions, each pointing to a counter in a different row of a 2D array, and increments each counter on every occurrence. To estimate frequency, it looks up the same positions and takes the minimum value. It uses the minimum because hash collisions with other items can only ever inflate a counter (never deflate it), so the smallest of several independent, possibly-inflated estimates is the one closest to the true value.

**7. Why is Count-Min Sketch's error described as "one-directional"?**
Because collisions between different items sharing a hash bucket can only cause a counter to be too high (overestimate), never too low — so a Count-Min Sketch estimate is either exactly correct or an overestimate, and never underestimates an item's true frequency.

**8. Scenario: You need to prevent processing the same payment transaction ID twice, out of billions processed per month, without storing every ID forever. Which structure fits, and what's the risk of using it alone?**
A Bloom filter fits as a fast, memory-efficient first check for "have I seen this ID before?" The risk of relying on it alone is its false-positive rate — it could occasionally say "seen before" for a transaction ID that's actually new, incorrectly skipping it. For something as critical as payment deduplication, it should be paired with an authoritative check (e.g., a database lookup) for any ID the filter flags as "maybe seen," rather than trusting the filter's answer outright when it says "yes."

**9. Scenario: A social media platform wants to display "trending hashtags" in near real-time out of millions of hashtags used per minute. Which structure fits, and why?**
Count-Min Sketch — the platform needs an approximate frequency count per hashtag cheaply enough to update on every single post, without maintaining an exact counter for every hashtag that will never be anywhere near trending. The one-directional overestimate error is an acceptable trade-off since the genuinely high-frequency hashtags will still clearly stand out above the noise.

**10. True or False: Probabilistic data structures like Bloom filters, HyperLogLog, and Count-Min Sketch are appropriate for tracking a bank account balance.**
False. These structures are designed for scenarios where an approximate answer produces an acceptable practical outcome. A bank account balance requires an exact, correct value every time — using a probabilistic structure here could introduce errors with real financial consequences, which is exactly the class of problem these tools are not meant for.
