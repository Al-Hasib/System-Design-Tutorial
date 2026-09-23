# ডায়াগ্রাম: একটি ভিডিও স্ট্রিমিং প্ল্যাটফর্ম ডিজাইন করা (YouTube/Netflix-এর মতো)

[README.md](README.md)-এর সাথে যুক্ত companion ডায়াগ্রাম।

## ১. সামগ্রিক আর্কিটেকচার (Overall Architecture)

Upload, transcoding, storage, metadata, এবং client player-এ CDN delivery।

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

*ক্যাপশন: Upload একটি queue-এর মাধ্যমে asynchronously distributed transcoding worker-দের মধ্যে প্রবাহিত হয়, অন্যদিকে playback read প্রায় সম্পূর্ণভাবে CDN edge cache থেকে সার্ভ করা হয়।*

## ২. Upload-থেকে-Playback-Ready সিকোয়েন্স

একজন ব্যবহারকারীর আপলোড থেকে ভিডিওটি streamable হয়ে ওঠা পর্যন্ত end-to-end pipeline।

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

*ক্যাপশন: Upload এবং transcoding path একটি queue দ্বারা asynchronous এবং decoupled, তাই playback readiness upload সময়ের নয়, বরং transcoding সময়ের সমান পিছিয়ে থাকে।*
</content>
