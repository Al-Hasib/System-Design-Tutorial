# ডায়াগ্রাম (Diagrams): Publish-Subscribe Pattern

## ১. মৌলিক Pub-Sub Fan-Out

```mermaid
flowchart LR
    Pub[Publisher] -->|publish event| Topic[(Topic: user-signed-up)]
    Topic --> S1[Email Service]
    Topic --> S2[Analytics Service]
    Topic --> S3[Loyalty Service]
```
*একটা publish করা event তিনজন স্বাধীন subscriber-এর কাছে fan out হয়, যাদের কারো সম্পর্কেই publisher অবগত নয়।*

## ২. Point-to-Point Queue বনাম Pub-Sub পাশাপাশি

```mermaid
flowchart TB
    subgraph PTP[Point-to-Point Queue]
        direction LR
        P1[Producer] --> Q1[(Queue)]
        Q1 --> C1[Consumer picks up ONE message]
    end
    subgraph PS[Publish-Subscribe]
        direction LR
        P2[Publisher] --> T1[(Topic)]
        T1 --> Sub1[Subscriber A]
        T1 --> Sub2[Subscriber B]
        T1 --> Sub3[Subscriber C]
    end
```
*একটা queue প্রতিটা message একজন মাত্র consumer-এর কাছে পৌঁছে দেয়; একটা topic প্রতিটা message প্রতিটা subscriber-এর কাছে পৌঁছে দেয়।*

## ৩. একটা Ride-Sharing Location Update Event-এর Sequence

```mermaid
sequenceDiagram
    participant Driver as Driver App
    participant Topic as location-updated Topic
    participant Map as Rider Map View
    participant ETA as ETA Service
    participant Fraud as Fraud Detection

    Driver->>Topic: publish location update
    Topic->>Map: deliver copy
    Topic->>ETA: deliver copy
    Topic->>Fraud: deliver copy
    Note over Map,Fraud: Each subscriber processes independently, in parallel
```
*একটা একক location update event স্বাধীনভাবে তিনটা অসম্পর্কিত downstream service-এ পৌঁছে দেওয়া হয়।*
</content>
