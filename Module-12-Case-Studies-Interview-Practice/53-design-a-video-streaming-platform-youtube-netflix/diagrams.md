# Diagrams: Design a Video Streaming Platform (like YouTube/Netflix)

Companion diagrams for [README.md](README.md).

## 1. Overall Architecture

Upload, transcoding, storage, metadata, and CDN delivery to the client player.

```mermaid
flowchart LR
    Client[Client Uploader]
    Upload[Upload Service]
    Raw[(Raw / Master Storage)]
    Queue[[Message Queue - Kafka]]
    Workers[Transcoding Workers]
    Encoded[(Encoded Storage - per-rendition segments)]
    Meta[(Metadata DB - sharded)]
    CDNOrigin[CDN Origin]
    Edge[CDN Edge PoPs]
    Player[Client Player - Adaptive Bitrate]

    Client -->|1. upload video| Upload
    Upload -->|2. store raw file| Raw
    Upload -->|3. publish upload-complete event| Queue
    Queue -->|4. consume job| Workers
    Workers -->|5. read raw file| Raw
    Workers -->|6. write renditions| Encoded
    Workers -->|7. update status: ready| Meta
    Encoded --> CDNOrigin
    CDNOrigin --> Edge
    Player -->|8. request manifest| Meta
    Player -->|9. request segments| Edge
    Edge -->|cache miss - pull from origin| CDNOrigin
```

*Caption: Uploads flow asynchronously through a queue into distributed transcoding workers, while playback reads are served almost entirely from CDN edge caches.*

## 2. Upload-to-Playback-Ready Sequence

The end-to-end pipeline from a user's upload to the video becoming streamable.

```mermaid
sequenceDiagram
    participant U as User
    participant UP as Upload Service
    participant RS as Raw Storage
    participant MQ as Message Queue
    participant TW as Transcoding Worker
    participant ES as Encoded Storage
    participant MD as Metadata Service
    participant CDN as CDN Edge
    participant P as Player (another user)

    U->>UP: Upload video (chunked, resumable)
    UP->>RS: Store raw master file
    UP->>MQ: Publish "video uploaded" event
    UP-->>U: Ack: processing started
    MQ->>TW: Deliver transcode job
    TW->>RS: Fetch raw file
    TW->>TW: Encode into renditions (240p...1080p/4K, segment into chunks)
    TW->>ES: Write encoded segments + manifest (HLS/DASH)
    TW->>MD: Update video status = ready, store rendition URLs
    P->>MD: Request video manifest
    MD-->>P: Return manifest (.m3u8 / .mpd)
    P->>CDN: Request first segment (lowest safe bitrate)
    CDN-->>P: Serve segment (cache hit) or fetch from ES on miss
    P->>P: Measure bandwidth, adapt bitrate per segment
```

*Caption: The upload and transcoding path is asynchronous and decoupled by a queue, so playback readiness lags upload by the transcoding time, not the upload time.*
