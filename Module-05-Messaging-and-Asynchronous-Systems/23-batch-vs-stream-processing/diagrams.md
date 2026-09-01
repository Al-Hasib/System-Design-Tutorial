# Diagrams: Batch vs Stream Processing

## 1. Batch Processing Flow

```mermaid
flowchart LR
    D1[Raw Data - collected over 24h] --> Store[(Data Lake / Warehouse)]
    Store -->|nightly job triggers| Job[Batch Job - e.g. Spark]
    Job --> Report[Daily Report / Model Update]
```
*Data accumulates over a bounded window before a single batch job processes it all at once.*

## 2. Stream Processing Flow with Windowing

```mermaid
flowchart LR
    E1[Event] --> Bus[(Stream / Kafka Topic)]
    Bus --> SP[Stream Processor - e.g. Flink]
    SP -->|tumbling 60s window| Agg[Real-Time Aggregate]
    Agg --> Alert[Live Dashboard / Alert]
```
*Events are processed continuously as they arrive, aggregated into short time windows for near real-time results.*

## 3. Lambda Architecture: Batch and Speed Layers Merged

```mermaid
flowchart TB
    Source[Incoming Data] --> Batch[Batch Layer - accurate, slow]
    Source --> Speed[Speed Layer - fast, approximate]
    Batch --> Serving[Serving Layer]
    Speed --> Serving
    Serving --> Query[Query Result: accurate + fresh]
```
*Lambda architecture runs data through both a slow accurate batch layer and a fast approximate speed layer, merging both views at query time.*
