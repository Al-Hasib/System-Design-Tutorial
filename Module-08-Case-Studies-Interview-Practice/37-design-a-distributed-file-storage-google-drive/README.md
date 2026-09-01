# Design a Distributed File Storage System (like Google Drive/Dropbox)

**Difficulty:** Advanced (Capstone)
**Estimated length:** 25-30 min
**Prerequisites:**
- [Design a Distributed File Storage System (like Google Drive/Dropbox)](../../Module-03-Databases-and-Storage/13-database-replication/README.md)
- [Design a Distributed File Storage System (like Google Drive/Dropbox)](../../Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md)
- [Design a Distributed File Storage System (like Google Drive/Dropbox)](../../Module-03-Databases-and-Storage/15-cap-theorem-and-pacelc/README.md)
- [Design a Distributed File Storage System (like Google Drive/Dropbox)](../../Module-04-Caching-and-Content-Delivery/18-cdn-explained/README.md)
- [Design a Distributed File Storage System (like Google Drive/Dropbox)](../../Module-05-Messaging-and-Asynchronous-Systems/22-event-driven-architecture/README.md)
- [Design a Distributed File Storage System (like Google Drive/Dropbox)](../../Module-06-Distributed-Systems-Concepts/27-consensus-algorithms-paxos-and-raft/README.md)
- [Design a Distributed File Storage System (like Google Drive/Dropbox)](../../Module-06-Distributed-Systems-Concepts/29-data-consistency-models-and-idempotency/README.md)

## Learning Objectives

- Decompose a cloud file-sync product into a **metadata service** and a **block/chunk storage service**, and explain why that separation is the core design decision.
- Estimate storage, bandwidth, and request-rate numbers for a service at Google Drive/Dropbox scale.
- Apply database sharding and replication (Module 3), consensus algorithms (Module 6), event-driven architecture (Module 5), and CDNs (Module 4) to concrete parts of the file storage problem.
- Explain chunking, content-addressable storage, and delta sync as the mechanisms that make large-file sync efficient.
- Reason about the trade-offs between strong and eventual consistency for metadata versus blob data, and about conflict resolution for offline edits.

## Script

### Hook/Intro

"You drop a 2 GB video into your Drive folder on your laptop. Twenty seconds later, it shows up on your phone, your tablet, and your teammate's laptop across the world -- without re-uploading the whole file if you tweak a single frame. How does that work at the scale of a billion users and exabytes of data? Today we're designing a distributed file storage and sync system -- think Google Drive, Dropbox, or OneDrive. This is a capstone problem: it pulls together sharding, replication, consensus, event-driven messaging, and CDNs from earlier modules into one system. Let's treat this like a real interview and work through it step by step."

### Step 1: Clarify Requirements

"Before designing anything, I'd nail down scope with the interviewer.

**Functional requirements:**
- Users can **upload and download** files of arbitrary size, from a 1 KB text file to a 50+ GB video archive.
- Files **sync automatically across devices** -- edit on your laptop, see the update on your phone within seconds.
- The system keeps **file version history**, so users can roll back to a previous version.
- Users can **share files and folders** with others, with read/write/owner permissions.
- Clients should support **offline edits** -- you can keep working without a network connection, and changes reconcile when you reconnect.

**Non-functional requirements:**
- **Durability** is the top priority -- we're quoting the industry-standard **99.999999999% (eleven nines)** annual durability. Losing a user's files is an unacceptable failure mode, even if the system is occasionally slow.
- **High availability** for uploads, downloads, and metadata reads -- targeting something like 99.9% availability.
- **Efficient handling of large files** via chunking -- we should never re-transfer an entire file for a one-byte change.
- **Bandwidth efficiency** via delta sync -- only the changed bytes should cross the network.
- We'll explicitly de-prioritize real-time collaborative co-editing (that's a different problem, closer to Google Docs' operational-transform/CRDT model) -- we're focused on file-level sync."

### Step 2: Capacity Estimation

"Let's put real numbers on this so our design decisions are grounded.

- **Users:** 500 million total users, with roughly 100 million daily active users (DAU).
- **Storage per user:** average 5 GB stored per user (free + paid tiers blended). That's `500M x 5GB = 2.5 exabytes` of total blob storage. With 11-nines durability we typically triple-replicate or erasure-code across data centers, so provisioned raw capacity is more like 3-5x that -- call it 8-12 exabytes.
- **Daily uploads:** if each DAU touches/uploads roughly 50 MB/day on average (mix of new files and edits), that's `100M x 50MB = 5 PB/day` of new/changed data ingested.
- **Chunking:** we split files into fixed-size **chunks of 4 MB**. A 2 GB file is ~500 chunks. This is the unit of storage, deduplication, and delta transfer.
- **Metadata records:** every file/folder is a metadata row, and every chunk reference is a row too. If the average user has 2,000 files, that's `500M x 2,000 = 1 trillion` file-metadata records, plus several trillion chunk-reference records mapping files to their constituent chunk hashes. Each metadata record is small -- maybe 200-500 bytes -- so total metadata volume is in the tens of terabytes, which is a very different scaling problem than the exabyte-scale blob data.
- **Request rates:** metadata operations (listing a folder, checking sync status, resolving permissions) are frequent and latency-sensitive -- estimate 500K-1M requests/sec globally at peak. Blob storage requests (actual chunk upload/download) are far less frequent per user but far heavier in bytes -- maybe 50K-100K requests/sec, but each moving megabytes.

This split -- **high QPS, small payloads for metadata** vs. **lower QPS, huge payloads for blobs** -- is exactly why we architect them as two separate services with very different scaling strategies."

### Step 3: High-Level Design

"At a high level, the flow looks like this:

1. **Sync client**: a background agent on the user's device watches a local folder, maintains a local embedded database (an index of file paths, chunk hashes, and sync state), and detects changes using filesystem events or periodic scans.
2. **API Gateway / Load Balancer**: the entry point for all client traffic, doing TLS termination, auth, and routing requests to the right backend service.
3. **Metadata Service**: owns the source of truth for the file/folder tree, versions, permissions, and the mapping from a file version to its list of chunk hashes. This is backed by a sharded, replicated database.
4. **Chunk/Block Storage Service**: an object store (think S3-like blob storage) that stores immutable chunks addressed by their content hash. This is where the exabytes actually live.
5. **Notification Service**: an event-driven pub/sub layer that tells a user's *other* devices, 'hey, file X changed, go sync it,' so updates propagate in near real time instead of relying on client polling.

The upload path: client chunks the file locally, hashes each chunk, asks the metadata service which chunks are already known (dedup check), uploads only the missing chunks to block storage, then commits the new file version (an ordered list of chunk hashes) to the metadata service. The metadata service then publishes a 'file changed' event, and the notification service fans that out to the user's other connected devices, which pull the delta."

### Step 4: Deep Dive on Key Components

"Let's zoom into the pieces that make this system actually work at scale.

**Chunking and content-addressable storage.** Every file is split into fixed-size chunks (~4 MB), and each chunk is hashed (e.g., SHA-256) to produce a content address. We store chunks keyed by that hash in the block storage service -- this is **content-addressable storage (CAS)**. Two enormous benefits fall out of this: (1) **deduplication** -- if two users upload the identical PDF, or a user uploads the same photo twice, the chunks are already present and we skip the upload entirely, which matters hugely at 2.5 exabytes of stated storage; and (2) **efficient sync** -- when a user edits a large file, only the chunks that actually changed get new hashes, so we upload and download just the delta, not the whole file. This is the same principle behind `rsync`-style delta encoding.

**Metadata database: sharding and replication (Module 3).** With a trillion-plus metadata rows and heavy read/write QPS, a single database can't hold this. We shard by **user ID** (or a `file_owner_id` for shared drives), so that all of a given user's files land on the same shard -- this keeps 'list my files' and 'sync my account' queries local to one shard instead of scattering across the cluster. Each shard is a replicated database (e.g., 3-5 replicas per shard) for durability and read scaling -- straight out of the replication playbook from Module 3.

**Consensus for metadata consistency (Module 6).** Losing or corrupting metadata is as bad as losing the file itself -- a dangling chunk reference makes a file unrecoverable. So writes to a metadata shard go through a **Raft (or Paxos)** consensus group: a write (like committing a new file version) is only acknowledged once a majority of replicas in that shard have durably applied it. This gives us strong consistency and automatic leader failover within a shard, which is critical since metadata is the 'source of truth' pointer into the blob store.

**Event-driven architecture for cross-device sync (Module 5).** After the metadata service commits a change, it publishes an event (`file.updated`, `file.shared`, `version.created`) to a message broker. The notification service subscribes to these events and pushes lightweight 'something changed, go check' pings to a user's other online devices over a persistent connection (WebSocket or long-poll). This pub/sub, event-driven approach means devices don't need to poll the metadata service constantly -- it's push-based, low-latency, and decouples 'detecting a change' from 'notifying every device.'

**CDN for fast downloads (Module 4).** For publicly shared files or files shared broadly within an organization, we front the block storage service with a **CDN**. A shared video or a company-wide onboarding PDF gets cached at edge locations close to viewers, so repeated downloads of the same popular chunk don't all hit origin storage -- this cuts latency and origin load dramatically for 'hot' shared content, while private, rarely-accessed files stay served directly from the object store."

### Step 5: Bottlenecks & Trade-offs

"No design is complete without naming its trade-offs.

- **Conflict resolution for concurrent/offline edits.** If a user edits a file on their laptop while offline, and also edits it on their phone, we get two divergent versions. We can't silently merge arbitrary binary files, so the common approach is **last-writer-wins with conflict copies** -- the system keeps both versions and creates a 'conflicted copy' file for the user to manually reconcile, similar to what Dropbox does today. We use file version numbers or vector clocks (Module 6 territory) to detect that a conflict occurred, rather than silently overwriting data.
- **Strong vs. eventual consistency (CAP theorem).** For metadata, we lean toward **strong consistency** (via Raft-backed writes) because an inconsistent file tree is confusing and can cause data loss. For blob/chunk storage, we lean toward **eventual consistency and availability** -- if a newly uploaded chunk takes an extra second to propagate to all replicas, that's an acceptable trade-off for higher write throughput and availability, since the chunk is immutable and content-addressed anyway (there's no 'stale value' problem, only a 'not yet replicated everywhere' problem).
- **Storage cost optimization.** Deduplication (via CAS) is our first lever. The second is **storage tiering**: files not accessed in, say, 90 days get moved to cheaper, higher-latency cold storage (like S3 Glacier-class tiers), while hot/recent files stay on fast storage. Version history is also a cost driver -- we cap the number of retained versions or their retention window to bound storage growth.
- **Resumable large uploads.** Because we upload chunk-by-chunk, a 50 GB upload over a flaky connection doesn't need to restart from zero on failure -- the client tracks which chunks have been acknowledged and only retries the missing ones. This chunk-level checkpointing is a direct, practical payoff of the chunking decision we made in Step 4."

### Recap

"To recap: we split the problem into a metadata service (sharded, replicated, consensus-backed for strong consistency) and a block storage service (content-addressable, deduplicated, eventually consistent, CDN-fronted for hot content). Chunking with content hashes gives us deduplication, delta sync, and resumable uploads all from one mechanism. Event-driven pub/sub keeps devices in sync in near real time without polling. And our trade-offs -- last-writer-wins with conflict copies, strong metadata / eventual blob consistency, and storage tiering -- reflect the actual choices production systems like Dropbox and Google Drive make."

### What's Next

"This wraps up our case-study series applying core distributed systems concepts to real products. From here, try sketching this design on your own for a variant -- how would your answer change for a system with real-time collaborative editing on top, like Google Docs? That's a great follow-up problem to practice next."

## Key Takeaways

- Separate the **metadata service** (small records, high QPS, strongly consistent, sharded by user/file ID with Raft-backed replication) from the **block storage service** (huge payloads, content-addressable, deduplicated, eventually consistent).
- **Chunking** (~4 MB chunks, content-hashed) is the single mechanism that enables deduplication, delta sync, and resumable uploads.
- **Consensus algorithms (Raft/Paxos)** protect metadata correctness; losing a chunk reference is as bad as losing the file.
- **Event-driven pub/sub** is what makes multi-device sync feel instant instead of poll-based.
- **CDNs** matter for shared/public file downloads, not for private per-user data.
- Trade-offs are asymmetric by design: strong consistency where correctness of the file tree matters, eventual consistency and availability where immutability of chunks makes it safe.
