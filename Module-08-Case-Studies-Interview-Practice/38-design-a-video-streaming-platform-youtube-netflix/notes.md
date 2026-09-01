# Notes: Design a Video Streaming Platform (like YouTube/Netflix)

Interview cheat-sheet companion to [README.md](README.md).

## Requirements

**Functional (in scope):**
- Upload video (resumable, chunked)
- Transcode into multiple resolutions/bitrates (adaptive bitrate renditions)
- Stream playback with adaptive bitrate streaming (HLS/DASH)
- Basic metadata: title, description, thumbnail, view count, likes

**Out of scope (call out explicitly):** search/ranking, recommendations, comments/social graph, live streaming ingest, monetization/ads.

**Non-functional:**
- High availability (playback is the core product)
- Low startup latency, minimal buffering
- Massive read-heavy scale (reads >> writes)
- Durability of stored video (near-zero data loss once accepted)

## Capacity Estimation (order-of-magnitude, YouTube-scale)

| Metric | Estimate |
|---|---|
| Video uploaded | ~500 hours/minute → ~30,000 video-hours/day |
| Raw upload volume | ~67 TB/day (avg ~5 Mbps raw bitrate) |
| Transcoding fan-out | ~5-6 renditions per video (240p-1080p/4K, multiple codecs) |
| Encoded storage growth | ~100-150 TB/day (in addition to retained raw masters) |
| Video views | ~5 billion views/day |
| Avg read QPS | ~58,000 views/sec average |
| Peak read QPS | ~150,000-250,000 concurrent stream starts/sec at peak |
| Avg stream bitrate | ~3 Mbps per concurrent session |
| Peak CDN egress | tens of Tbps to ~150 Tbps at global peak (scale with concurrent viewers) |

## Architecture Summary

```
Client (upload) -> Upload Service -> Raw/Master Storage (object store)
                                        |
                                        v
                                Message Queue (Kafka)
                                        |
                                        v
                          Transcoding Workers (distributed, horizontally scaled)
                                        |
                                        v
                          Encoded Storage (object store, per-rendition segments)
                                        |
                        +---------------+----------------+
                        v                                v
                Metadata Service (sharded DB)      CDN Origin
                                                          |
                                                          v
                                                   CDN Edge PoPs
                                                          |
                                                          v
                                        Client Player (adaptive bitrate: HLS/DASH)
```

Write path: Upload -> Raw Storage -> Queue -> Transcoding Workers -> Encoded Storage -> Metadata update.
Read path: Client -> Metadata/API (manifest) -> CDN Edge (cache hit/miss to origin) -> Player.

## Key Decisions & Trade-offs

| Decision | Option A | Option B | Notes |
|---|---|---|---|
| Streaming protocol | HLS | MPEG-DASH | HLS is Apple-originated, universal on iOS/Safari, uses `.m3u8` manifests + `.ts`/fMP4 segments. DASH is an open, codec-agnostic ISO standard, widely used on Android/web. Many platforms serve both from the same encoded segments to maximize device compatibility. |
| Transcoding timing | Pre-transcode all renditions on upload | Transcode top renditions on upload, generate rare ones (4K, extra codecs) on-demand and cache | Pre-transcode-all is simple but wastes compute on rarely-viewed renditions. On-demand lazily saves cost but adds latency on first request for a rare rendition. |
| CDN caching strategy | Cache-aside, TTL + content-versioned URLs for all content uniformly | Tiered: aggressively cache/replicate hot & viral content at all edges; keep long-tail content in cheaper, colder storage with edge fetch-on-demand | Popular videos justify wide edge replication; long-tail videos (most of the catalog, few views each) don't — tiering saves significant storage/CDN cost. |
| View counter writes | Synchronous increment on primary metadata shard per view | Stream-aggregate view events (windowed) and periodically flush aggregated counts | Synchronous writes create hot-shard contention on viral videos; aggregation trades small staleness for much lower write amplification. |
| Video codec | H.264 only (max compatibility) | H.264 + AV1/VP9 (better compression, newer devices) | Dual-encoding raises transcoding cost but cuts long-term storage/egress cost and improves quality-per-bit on supporting devices. |

## Bottlenecks to Mention

- Transcoding cost/latency vs. pre-generating all renditions
- Hot vs. long-tail content caching and storage cost
- Thundering herd / cache stampede on newly viral videos (mitigate with request coalescing, pre-warming, origin shield)
- Codec/format trade-offs (compression efficiency vs. encode cost vs. device support)
