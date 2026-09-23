# Notes: Distributed File Storage System (Google Drive/Dropbox)

দ্রুত interview cheat-sheet। `README.md` (পূর্ণ script), `diagrams.md`, এবং `quiz.md`-এর সাথে মিলিয়ে ব্যবহার করুন।

## Requirements Summary

**Functional**
- যেকোনো size-এর file upload/download (KB থেকে 50+ GB)
- একাধিক device জুড়ে স্বয়ংক্রিয়ভাবে sync
- File version history / rollback
- read/write/owner permission সহ sharing
- পরে reconciliation সহ offline edit

**Non-functional**
- Durability: 99.999999999% (11 nines) -- সবচেয়ে বড় অগ্রাধিকার
- Availability: ~99.9%
- Chunking-এর মাধ্যমে বড় file সমর্থন
- Delta sync-এর মাধ্যমে bandwidth efficiency
- Scope-এর বাইরে: real-time co-editing (এটি Google Docs / CRDT সমস্যা)

## Capacity সংখ্যা (order-of-magnitude)

| Metric | Estimate |
|---|---|
| মোট user | 500M (100M DAU) |
| User প্রতি গড় storage | 5 GB |
| মোট blob storage | ~2.5 exabytes (logical); replication/erasure coding সহ ~8-12 EB raw |
| দৈনিক ingest হওয়া data | ~5 PB/day (100M DAU x ~50MB/day) |
| Chunk size | 4 MB |
| File-metadata record | ~1 trillion (500M user x ~2,000 file) |
| Metadata QPS | 500K-1M req/sec (ছোট payload) |
| Blob storage QPS | 50K-100K req/sec (বড় payload) |

## Architecture Summary

1. **Sync client** -- file + chunk hash + sync state-এর local DB/index; পরিবর্তনের জন্য filesystem watch করে।
2. **API Gateway / Load Balancer** -- entry point, auth, routing।
3. **Metadata Service** -- file tree, version, permission, chunk-hash list-এর source of truth। User ID (বা owner ID) দিয়ে sharded, প্রতিটি shard strongly consistent write-এর জন্য Raft/Paxos consensus দিয়ে replicated।
4. **Block/Chunk Storage Service** -- content-addressable object store; chunk-গুলো content hash (যেমন, SHA-256) দিয়ে keyed; deduplicated; eventually consistent replication।
5. **Notification Service** -- event-driven pub/sub; near-real-time sync-এর জন্য user-এর অন্যান্য online device-এ "file changed" event push করে।
6. **CDN** -- hot content-এ latency এবং origin load কমাতে shared/public file-এর জন্য block storage-এর সামনে বসে।

**Upload path:** client file chunk করে -> chunk হ্যাশ করে -> metadata service-কে জিজ্ঞেস করে কোন hash ইতিমধ্যে আছে (dedup check) -> শুধু missing chunk upload করে -> নতুন file version (ordered chunk-hash list) metadata service-এ commit করে -> metadata service `file.updated` event publish করে -> notification service অন্যান্য device-এ fan out করে -> সেসব device শুধু missing/পরিবর্তিত chunk pull করে (delta sync)।

## মূল সিদ্ধান্ত ও Trade-off

| সিদ্ধান্ত | Option A | Option B | পছন্দ ও কারণ |
|---|---|---|---|
| Storage granularity | Full-file storage (প্রতিটি file একটি blob হিসেবে store) | Block/chunk storage (fixed-size, content-hashed chunk-এ ভাগ) | **Block storage।** Dedup, delta sync, এবং resumable upload সম্ভব করে; full-file storage যেকোনো edit-এ পুরো file আবার transfer/store করে। |
| Metadata consistency | Strong consistency (consensus-backed write) | Eventual consistency | Metadata-র জন্য **Strong** -- একটি inconsistent file tree / dangling chunk reference data loss বা বিভ্রান্তি ঘটায়; write Raft/Paxos majority ack-এর মধ্য দিয়ে যায়। |
| Blob/chunk consistency | Strong consistency | Eventual consistency | Chunk replication-এর জন্য **Eventual** -- chunk immutable এবং content-addressed, তাই "stale value" কোনো ঝুঁকি নয়, শুধু replication lag; availability এবং write throughput-কে অগ্রাধিকার দেয় (CAP/PACELC trade-off)। |
| Conflict handling | Auto-merge | Last-writer-wins + conflicted copy | **Conflicted copy।** নির্বিচারে binary file নিরাপদে auto-merge করা যায় না; উভয় version রাখুন এবং user-কে resolve করতে দিন, যেমন Dropbox করে। |
| Cold data | সবকিছু hot storage-এ রাখা | নিষ্ক্রিয়তার পর cold storage-এ tier করা | **Tiering।** ~90 দিন access না হওয়া file সস্তা cold/archival storage-এ move করুন; cost নিয়ন্ত্রণ করতে version-history retention সীমিত করুন। |
| Public/shared download | সরাসরি object store থেকে serve | CDN সামনে বসানো | Shared/public file-এর জন্য **CDN** -- hot content-এর জন্য latency এবং origin load কমায়; private file সরাসরি object store থেকেই serve হতে থাকে। |

## ধারাবাহিকভাবে ব্যবহৃত পরিভাষা

- **Sync client** -- device-side background agent
- **Metadata service** -- file tree, version, permission, chunk-hash list (sharded + replicated + Raft/Paxos)
- **Block/chunk storage service** (ওরফে object store) -- content-addressable, deduplicated blob storage
- **Chunk** -- একটি file-এর fixed-size (4 MB) content-hashed একক
- **Notification service** -- cross-device sync-এর জন্য event-driven pub/sub
- **Delta sync** -- পুরো file নয়, শুধু পরিবর্তিত chunk transfer করা
- **Content-addressable storage (CAS)** -- chunk-গুলো তাদের content hash দিয়ে keyed/addressed
