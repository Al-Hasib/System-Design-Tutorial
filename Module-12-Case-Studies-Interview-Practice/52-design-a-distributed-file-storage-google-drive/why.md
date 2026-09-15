# Why This Topic Matters: Design a Distributed File Storage System (like Google Drive/Dropbox)

> **In one sentence:** This is the case study where metadata and content must be separated, where a one-byte edit must not re-upload a gigabyte, and where two people editing the same file offline forces you to answer a conflict question that has no clean technical solution.

## Why This Case Study Exists

Most design problems deal in small records. This one deals in large, opaque binary objects — and that changes everything about storage, transfer, and consistency.

It is also the clearest example in the course of a system where **the metadata problem and the data problem are entirely different problems**. File metadata (names, folder structure, permissions, versions, sharing) is small, highly relational, frequently queried, and needs transactions. File content is enormous, immutable once written, and needs cheap durable storage and fast delivery. Trying to serve both with one system produces something that is bad at both — recognizing that split early is the first real insight of the problem.

## The Design Problems It Forces You to Solve

### 1. Uploading and syncing large files efficiently
**The problem:** A user edits one paragraph in a 2 GB video project. Re-uploading 2 GB is unacceptable. A 5 GB upload over a flaky connection must not restart from zero.

**Why it is hard:** Files are opaque blobs; naive systems treat them as atomic.

**What you learn:** **Chunking.** Split files into fixed or content-defined blocks (4 MB is a common choice), hash each one, and treat the file as an ordered list of chunk hashes. Now a small edit uploads one chunk. An interrupted upload resumes from the last acknowledged chunk. Sync compares hash lists and transfers only the difference. This single technique is what makes the product feasible, and content-defined chunking (boundaries chosen by content, not offset) is the refinement that keeps edits from shifting every subsequent boundary.

### 2. Storing the same content once
**The problem:** Ten thousand employees have the same onboarding PDF. A user uploads a file they already have in another folder.

**What you learn:** **Content-addressed deduplication** — store each chunk under its hash, and a second upload of identical content is just a reference. This is a large real-world cost saving, and it comes with consequences worth naming: deletion now requires reference counting, and cross-user deduplication has a privacy side channel (you can learn whether a file already exists by observing upload speed), which is why some systems deduplicate only within a user's own account.

### 3. Separating metadata from content
**The problem:** Listing a folder must be instant; storing a petabyte must be cheap.

**What you learn:** Metadata goes in a database — relational is a good fit, because the folder hierarchy, sharing permissions, and version history are genuinely relational and benefit from transactions. Content goes in object storage (S3-style), which is built for durability and cheap bulk capacity. The client talks to the metadata service to learn *what* the file is, and then uploads or downloads chunks directly to and from object storage using a presigned URL — **keeping the bytes off your application servers entirely**, which is the detail that separates a workable design from one where your API tier becomes a bandwidth bottleneck.

### 4. Conflicts when two devices edit offline
**The problem:** A user edits a document on a laptop with no connectivity while a colleague edits the same file elsewhere. Both come online.

**Why it is hard:** There is no technically correct merge for arbitrary binary content, and choosing "last write wins" on wall-clock timestamps silently destroys someone's work based on clock skew.

**What you learn:** That this is a **product decision expressed as an architecture**. Version vectors detect that the two edits were concurrent rather than sequential; what you do next — keep both as "Document (conflicted copy)", prompt the user, or merge for known formats — is a choice about user experience. Being explicit that detection and resolution are separate, and that detection needs logical clocks rather than timestamps, is the high-signal answer here.

### 5. Propagating changes to every device
**The problem:** A change on one device should appear on the user's other devices and on collaborators' devices quickly, without every client polling constantly.

**What you learn:** A notification channel (long-lived connection or push) that tells clients "something changed, come fetch the delta," combined with a per-user monotonically increasing change cursor so clients can ask "what has changed since sequence 4821?" This is far more efficient than diffing whole trees and is how real sync engines work.

## What It Costs to Get Wrong

- **Treating files as atomic blobs** means no resumable uploads, no delta sync, and a product that is unusable on real networks.
- **Routing file bytes through your API servers** turns your application tier into a bandwidth-bound cost center; presigned direct-to-object-storage is the standard fix.
- **Putting file content in a relational database** is the classic wrong answer and is expensive in every dimension.
- **Resolving conflicts by timestamp** loses user data silently — the one failure mode users never forgive.
- **Forgetting reference counting** with deduplication means deleting one user's file destroys another user's data.

## Why Interviewers Choose This One

It moves the conversation away from request/response throughput and into storage architecture, data transfer efficiency, and offline-first synchronization — a distinctly different skill set from the feed and chat problems. It also has an unusually clean layered structure (client sync engine, metadata service, block storage, notification service), which makes it a good test of whether a candidate can decompose a system into services with clear responsibilities. And the conflict question reliably separates candidates who reach for a timestamp from those who know why that is unsafe.

## How It Connects

This case study applies **object storage and SQL/NoSQL selection** for the metadata-versus-content split (topic 11), **replication and durability** (topic 13), **sharding** of metadata by user (topic 14), **CDN** for download acceleration (topic 18), **logical clocks and vector clocks** for conflict detection (topic 41), **idempotency** for resumable chunk uploads (topic 29), **queues** for asynchronous processing like thumbnailing and virus scanning (topic 20), **WebSockets or push** for change notification (topic 10), and **security** for encryption at rest and sharing permissions (topic 36).

**Next:** [Design a Video Streaming Platform](../53-design-a-video-streaming-platform-youtube-netflix/why.md) — where the content is even larger and the delivery network becomes the product.
