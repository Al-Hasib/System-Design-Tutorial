# Diagrams: Distributed File Storage System (Google Drive/Dropbox)

## 1. Overall Architecture

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

*Caption: The metadata service (strongly consistent, sharded/replicated) and block storage service (deduplicated, eventually consistent) are separate services; the notification service pushes change events so other devices sync without polling, and a CDN fronts hot shared-file downloads.*

## 2. Sequence: Uploading and Syncing a Changed File to Another Device

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

*Caption: Only missing/changed chunks are ever transferred -- both on initial upload (dedup) and on sync to the second device (delta sync) -- while the notification service makes the update visible on device B in near real time.*
