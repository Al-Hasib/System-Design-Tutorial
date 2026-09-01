# Quiz: Follow-up Interview Questions

Practice questions an interviewer might ask after the main design. Model answers are intentionally concise -- expand on them out loud in a real interview.

**1. How would you handle two devices editing the same file concurrently, especially if one was offline?**
Detect the conflict using file version numbers (or vector clocks) tracked by the metadata service -- if a device tries to commit a new version whose "based on" version no longer matches the current head, that's a conflict. Rather than attempting an unsafe automatic merge of arbitrary binary content, keep both versions: apply the winning write (e.g., last-writer-wins by timestamp) as the canonical file, and save the other as a "conflicted copy" for the user to manually reconcile, similar to Dropbox's approach.

**2. How do you implement efficient delta sync so you don't re-upload a whole file after a small edit?**
Split files into fixed-size chunks (e.g., 4 MB) and hash each chunk. On edit, only the chunks whose content actually changed get new hashes; the client diffs its local chunk-hash list against the last-synced version and uploads/downloads only the chunks that differ. This is the same principle as rsync's rolling-checksum delta encoding, adapted to fixed-size, content-addressed chunks.

**3. How do you deduplicate identical files uploaded by different users?**
Because chunks are stored in a content-addressable store keyed by their hash, if two users upload byte-identical files (or files sharing common chunks), the chunk hashes match existing entries and the upload is skipped entirely -- the metadata service just adds a reference to the existing chunks. This works across users transparently since the store is global, not per-user.

**4. How would you handle uploading a 50 GB file over an unreliable network connection?**
Chunk the file client-side and upload chunks independently with per-chunk acknowledgments. The client persists which chunks have been acked; on disconnect/retry, it resumes from the last unacknowledged chunk instead of restarting the whole transfer. The final "commit" step (writing the ordered chunk-hash list to the metadata service) only happens after all chunks are confirmed stored, so a partial upload never becomes a corrupt "complete" file.

**5. How would you design the sharing/permissions model?**
Store an access-control list (ACL) per file/folder in the metadata service: a list of (principal, role) pairs where role is owner/editor/viewer. For folder shares, permissions are inherited down the tree unless explicitly overridden, resolved at read time by walking up to the nearest explicit grant. Permission checks happen in the metadata service before returning chunk-hash lists or granting upload commit rights, so blob storage itself stays permission-agnostic.

**6. How do you manage file version history without storage costs exploding?**
Because storage is chunk-based, a new version typically only adds a small number of new chunks (the changed parts) while reusing unchanged chunks from prior versions via reference counting -- so storing N versions costs far less than N full copies. On top of that, apply retention policies: cap the number of retained versions or their retention window (e.g., 30-180 days), and let paid tiers keep longer history.

**7. What's the trade-off between strong and eventual consistency in this system, and where do you apply each?**
Per CAP/PACELC: for metadata (the file tree, permissions, "what is the current version"), we choose strong consistency via Raft/Paxos-backed writes, because an inconsistent view of the file tree can mean data loss or exposing stale permissions. For blob/chunk storage, we choose eventual consistency, because chunks are immutable and content-addressed -- there's no "stale value" to read, only replication lag before a chunk is available everywhere -- so we favor availability and write throughput instead.

**8. How would you reduce storage costs at exabyte scale?**
Three levers: (1) deduplication via content-addressable chunk storage, which eliminates redundant copies of identical content; (2) storage tiering, moving files not accessed for ~90 days to cheaper cold/archival storage classes; (3) bounding version-history retention so old, superseded chunks can eventually be garbage-collected once no live version references them.

**9. How does a client know when to fetch updates instead of polling constantly?**
The metadata service publishes an event (e.g., `file.updated`) whenever a version is committed, using an event-driven pub/sub architecture. The notification service subscribes and pushes a lightweight "something changed, go check metadata" signal over a persistent connection (WebSocket/long-poll) to the user's other online devices, which then pull just the metadata diff and the missing chunks. This is push-based and near-real-time, far cheaper than constant polling.

**10. Why not just replicate every chunk synchronously to every region before acknowledging an upload?**
That would maximize durability guarantees per write but tank availability and latency -- a slow or unreachable replica would block every upload. Instead, acknowledge after a quorum write (e.g., 2 of 3 replicas) for durability, and replicate asynchronously to remaining regions/replicas, accepting brief eventual consistency for the blob layer in exchange for availability and throughput -- consistent with treating chunk storage as the "AP-leaning" side of the system.

**11. How would you support very large folders (e.g., 100,000 files) without slow "list folder" operations?**
Paginate folder listings in the metadata service and index by (parent_folder_id, name) with a covering index, so a listing query is a bounded range scan on one shard (since the shard key is user/owner ID, the whole folder tree for a user typically lives on one shard). Avoid returning full chunk-hash lists in a listing call -- only return them when a specific file is opened/synced.

**12. How would this design change if you added real-time collaborative co-editing (like Google Docs)?**
File-level chunk sync is too coarse for character-by-character concurrent edits. You'd need an operation-based model -- Operational Transformation (OT) or CRDTs -- where the unit of sync is an edit operation, not a file chunk, and a central sequencing service (or peer-to-peer CRDT merge) resolves concurrent operations automatically instead of falling back to conflicted copies. The metadata/versioning and storage-tiering pieces of this design would still apply for the underlying document snapshots and history.
