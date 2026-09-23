# Diagrams: Distributed File Storage System (Google Drive/Dropbox)

## ১. সামগ্রিক Architecture

```mermaid
flowchart LR
    subgraph Devices
        SC1[Sync Client - Laptop]
        SC2[Sync Client - Phone]
    end

    SC1 -->|upload/download| LB[API Gateway / Load Balancer]
    SC2 -->|upload/download| LB

    LB --> MS[Metadata Service]
    LB --> BS[Block/Chunk Storage Service]

    MS --> MDB[(Sharded + Replicated Metadata DB<br/>Raft/Paxos consensus per shard)]
    BS --> OBJ[(Content-Addressable Object Store<br/>deduplicated chunks)]

    MS -->|publishes file.updated events| NS[Notification Service]
    NS -->|push: go sync| SC1
    NS -->|push: go sync| SC2

    CDN[CDN] --- OBJ
    Viewer[Shared-link Viewer] -->|download shared file| CDN
```

*Caption: metadata service (strongly consistent, sharded/replicated) এবং block storage service (deduplicated, eventually consistent) আলাদা service; notification service change event push করে যাতে অন্যান্য device polling ছাড়াই sync করে, এবং একটি CDN hot shared-file download-এর সামনে বসে।*

## ২. Sequence: একটি পরিবর্তিত File Upload এবং অন্য Device-এ Sync করা

```mermaid
sequenceDiagram
    participant A as Sync Client A (Laptop)
    participant GW as API Gateway
    participant MS as Metadata Service
    participant BS as Block Storage Service
    participant NS as Notification Service
    participant B as Sync Client B (Phone)

    A->>A: Detect file change, split into 4MB chunks, hash each chunk
    A->>GW: Which chunk hashes already exist? (dedup check)
    GW->>MS: Query known chunk hashes
    MS-->>GW: List of missing hashes
    GW-->>A: Upload only missing chunks

    A->>GW: Upload missing chunks
    GW->>BS: Store chunks (content-addressed)
    BS-->>GW: Chunks stored (ack)

    A->>GW: Commit new file version (ordered chunk-hash list)
    GW->>MS: Write new version (Raft-backed majority ack)
    MS-->>GW: Version committed
    MS->>NS: Publish file.updated event

    NS->>B: Push "file changed" notification
    B->>GW: Request updated chunk-hash list for file
    GW->>MS: Fetch latest version metadata
    MS-->>B: Ordered chunk-hash list
    B->>GW: Download only the changed chunks (delta sync)
    GW->>BS: Fetch missing chunks
    BS-->>B: Chunk data
    B->>B: Reassemble file locally
```

*Caption: initial upload (dedup) এবং দ্বিতীয় device-এ sync (delta sync) -- উভয় ক্ষেত্রেই শুধুমাত্র missing/পরিবর্তিত chunk transfer হয় -- আর notification service device B-তে near real time-এ update দৃশ্যমান করে তোলে।*
