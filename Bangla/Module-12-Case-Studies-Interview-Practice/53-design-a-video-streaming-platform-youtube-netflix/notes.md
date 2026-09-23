# নোটস: একটি ভিডিও স্ট্রিমিং প্ল্যাটফর্ম ডিজাইন করা (YouTube/Netflix-এর মতো)

[README.md](README.md)-এর সাথে যুক্ত ইন্টারভিউ চিট-শিট।

## Requirements

**Functional (scope-এর মধ্যে):**
- ভিডিও আপলোড (resumable, chunked)
- একাধিক resolution/bitrate-এ transcode (adaptive bitrate rendition)
- Adaptive bitrate streaming (HLS/DASH) সহ স্ট্রিম প্লেব্যাক
- মৌলিক metadata: title, description, thumbnail, view count, likes

**Scope-এর বাইরে (স্পষ্টভাবে উল্লেখ করুন):** search/ranking, recommendation, comments/social graph, live streaming ingest, monetization/ads।

**Non-functional:**
- High availability (প্লেব্যাকই মূল প্রোডাক্ট)
- কম startup latency, সর্বনিম্ন buffering
- বিশাল read-heavy স্কেল (reads >> writes)
- সংরক্ষিত ভিডিওর Durability (একবার গৃহীত হলে প্রায় শূন্য data loss)

## Capacity Estimation (order-of-magnitude, YouTube-স্কেলে)

| মেট্রিক | অনুমান |
|---|---|
| আপলোড হওয়া ভিডিও | ~500 ঘণ্টা/মিনিট → ~30,000 video-hours/day |
| Raw upload volume | ~67 TB/day (গড় ~5 Mbps raw bitrate) |
| Transcoding fan-out | প্রতি ভিডিওতে ~5-6টি rendition (240p-1080p/4K, একাধিক codec) |
| Encoded storage বৃদ্ধি | ~100-150 TB/day (ধরে রাখা raw master ছাড়াও) |
| ভিডিও view | ~5 বিলিয়ন views/day |
| গড় read QPS | ~58,000 views/sec গড়ে |
| Peak read QPS | peak-এ ~150,000-250,000 concurrent stream starts/sec |
| গড় stream bitrate | প্রতি concurrent সেশনে ~3 Mbps |
| Peak CDN egress | global peak-এ কয়েক দশ Tbps থেকে ~150 Tbps (concurrent viewer সংখ্যার সাথে স্কেল করে) |

## Architecture সারসংক্ষেপ

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

Write path: Upload -> Raw Storage -> Queue -> Transcoding Workers -> Encoded Storage -> Metadata update।
Read path: Client -> Metadata/API (manifest) -> CDN Edge (cache hit/miss to origin) -> Player।

## মূল সিদ্ধান্ত ও Trade-off

| সিদ্ধান্ত | অপশন A | অপশন B | নোট |
|---|---|---|---|
| Streaming protocol | HLS | MPEG-DASH | HLS Apple-উদ্ভূত, iOS/Safari-তে সার্বজনীন, `.m3u8` manifest + `.ts`/fMP4 segment ব্যবহার করে। DASH একটি open, codec-agnostic ISO স্ট্যান্ডার্ড, Android/web-এ ব্যাপকভাবে ব্যবহৃত। অনেক প্ল্যাটফর্ম ডিভাইস compatibility সর্বাধিক করতে একই encoded segment থেকে উভয়ই সার্ভ করে। |
| Transcoding timing | আপলোডের সময় সব rendition pre-transcode করা | আপলোডের সময় শীর্ষ rendition transcode করা, বিরল গুলো (4K, extra codec) on-demand তৈরি এবং cache করা | সব pre-transcode করা সহজ কিন্তু কদাচিৎ-দেখা rendition-এ compute অপচয় করে। On-demand lazily cost বাঁচায় কিন্তু বিরল rendition-এর প্রথম অনুরোধে latency যোগ করে। |
| CDN caching strategy | Cache-aside, সব কনটেন্টের জন্য একইভাবে TTL + content-versioned URL | Tiered: hot ও viral কনটেন্ট সব edge-এ আক্রমণাত্মকভাবে cache/replicate করা; long-tail কনটেন্ট সস্তা, ঠান্ডা storage-এ edge fetch-on-demand সহ রাখা | জনপ্রিয় ভিডিও ব্যাপক edge replication যুক্তিসঙ্গত করে; long-tail ভিডিও (catalog-এর বেশিরভাগ, প্রতিটির view কম) তা করে না — tiering উল্লেখযোগ্য storage/CDN cost বাঁচায়। |
| View counter write | প্রতি view-তে primary metadata shard-এ synchronous increment | View event stream-aggregate করা (windowed) এবং periodically aggregated count flush করা | Synchronous write viral ভিডিওতে hot-shard contention তৈরি করে; aggregation সামান্য staleness-এর বিনিময়ে অনেক কম write amplification দেয়। |
| Video codec | শুধু H.264 (সর্বাধিক compatibility) | H.264 + AV1/VP9 (ভালো compression, নতুন ডিভাইস) | Dual-encoding transcoding cost বাড়ায় কিন্তু দীর্ঘমেয়াদী storage/egress cost কমায় এবং সাপোর্ট করা ডিভাইসে quality-per-bit উন্নত করে। |

## উল্লেখ করার মতো Bottleneck

- সব rendition pre-generate করার তুলনায় Transcoding cost/latency
- Hot বনাম long-tail কনটেন্ট caching এবং storage cost
- নতুন viral ভিডিওতে Thundering herd / cache stampede (request coalescing, pre-warming, origin shield দিয়ে প্রতিকার)
- Codec/format trade-off (compression efficiency বনাম encode cost বনাম device support)
</content>
