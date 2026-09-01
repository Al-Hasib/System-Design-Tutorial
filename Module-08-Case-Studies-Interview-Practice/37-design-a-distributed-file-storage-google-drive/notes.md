# Notes: Distributed File Storage System (Google Drive/Dropbox)

Quick interview cheat-sheet. Pairs with `README.md` (full script), `diagrams.md`, and `quiz.md`.

## Requirements Summary

**Functional**
- Upload/download files of any size (KB to 50+ GB)
- Sync across multiple devices automatically
- File version history / rollback
- Sharing with read/write/owner permissions
- Offline edits with later reconciliation

**Non-functional**
- Durability: 99.999999999% (11 nines) -- top priority
- Availability: ~99.9%
- Large-file support via chunking
- Bandwidth efficiency via delta sync
- Out of scope: real-time co-editing (that's the Google Docs / CRDT problem)

## Capacity Numbers (order-of-magnitude)

| Metric | Estimate |
|---|---|
| Total users | 500M (100M DAU) |
| Avg storage/user | 5 GB |
| Total blob storage | ~2.5 exabytes (logical); ~8-12 EB raw with replication/erasure coding |
| Daily data ingested | ~5 PB/day (100M DAU x ~50MB/day) |
| Chunk size | 4 MB |
| File-metadata records | ~1 trillion (500M users x ~2,000 files) |
| Metadata QPS | 500K-1M req/sec (small payloads) |
| Blob storage QPS | 50K-100K req/sec (large payloads) |

## Architecture Summary

1. **Sync client** -- local DB/index of files + chunk hashes + sync state; watches filesystem for changes.
2. **API Gateway / Load Balancer** -- entry point, auth, routing.
3. **Metadata Service** -- source of truth for file tree, versions, permissions, chunk-hash lists. Sharded by user ID (or owner ID), each shard replicated with Raft/Paxos consensus for strongly consistent writes.
4. **Block/Chunk Storage Service** -- content-addressable object store; chunks keyed by content hash (e.g., SHA-256); deduplicated; eventually consistent replication.
5. **Notification Service** -- event-driven pub/sub; pushes "file changed" events to a user's other online devices for near-real-time sync.
6. **CDN** -- fronts block storage for shared/public files to cut latency and origin load on hot content.

**Upload path:** client chunks file -> hashes chunks -> ask metadata service which hashes already exist (dedup check) -> upload only missing chunks -> commit new file version (ordered chunk-hash list) to metadata service -> metadata service publishes `file.updated` event -> notification service fans out to other devices -> those devices pull only the missing/changed chunks (delta sync).

## Key Decisions & Trade-offs

| Decision | Option A | Option B | Choice & Why |
|---|---|---|---|
| Storage granularity | Full-file storage (store each file as one blob) | Block/chunk storage (split into fixed-size, content-hashed chunks) | **Block storage.** Enables dedup, delta sync, and resumable uploads; full-file storage re-transfers/re-stores entire files on any edit. |
| Metadata consistency | Strong consistency (consensus-backed writes) | Eventual consistency | **Strong** for metadata -- an inconsistent file tree / dangling chunk reference causes data loss or confusion; writes go through Raft/Paxos majority ack. |
| Blob/chunk consistency | Strong consistency | Eventual consistency | **Eventual** for chunk replication -- chunks are immutable and content-addressed, so "stale value" isn't a risk, only replication lag; favors availability and write throughput (CAP/PACELC trade-off). |
| Conflict handling | Auto-merge | Last-writer-wins + conflicted copy | **Conflicted copy.** Arbitrary binary files can't be safely auto-merged; keep both versions and let the user resolve, like Dropbox does. |
| Cold data | Keep everything on hot storage | Tier to cold storage after inactivity | **Tiering.** Move files unaccessed for ~90 days to cheaper cold/archival storage; bound version-history retention to control cost. |
| Public/shared downloads | Serve directly from object store | Front with a CDN | **CDN** for shared/public files -- cuts latency and origin load for hot content; private files stay served from the object store directly. |

## Terminology Used Consistently

- **Sync client** -- the device-side background agent
- **Metadata service** -- file tree, versions, permissions, chunk-hash lists (sharded + replicated + Raft/Paxos)
- **Block/chunk storage service** (a.k.a. object store) -- content-addressable, deduplicated blob storage
- **Chunk** -- fixed-size (4 MB) content-hashed unit of a file
- **Notification service** -- event-driven pub/sub for cross-device sync
- **Delta sync** -- transferring only changed chunks, not the whole file
- **Content-addressable storage (CAS)** -- chunks keyed/addressed by their content hash
