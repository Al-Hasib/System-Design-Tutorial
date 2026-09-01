# Diagrams: Publish-Subscribe Pattern

## 1. Basic Pub-Sub Fan-Out

```mermaid
flowchart LR
    Pub[Publisher] -->|publish event| Topic[(Topic: user-signed-up)]
    Topic --> S1[Email Service]
    Topic --> S2[Analytics Service]
    Topic --> S3[Loyalty Service]
```
*One published event fans out to three independent subscribers, none of whom the publisher is aware of.*

## 2. Point-to-Point Queue vs Pub-Sub Side by Side

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
*A queue delivers each message to a single consumer; a topic delivers each message to every subscriber.*

## 3. Sequence of a Ride-Sharing Location Update Event

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
*A single location update event is independently delivered to three unrelated downstream services.*
