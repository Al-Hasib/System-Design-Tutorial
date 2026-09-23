# Diagrams: Batch বনাম Stream Processing

## ১. Batch Processing Flow

```mermaid
flowchart LR
    D1[Raw Data - collected over 24h] --> Store[(Data Lake / Warehouse)]
    Store -->|nightly job triggers| Job[Batch Job - e.g. Spark]
    Job --> Report[Daily Report / Model Update]
```
*একটা bounded window ধরে ডেটা জমা হয়, তারপর একটা single batch job একবারে সবকিছু প্রসেস করে।*

## ২. Windowing সহ Stream Processing Flow

```mermaid
flowchart LR
    E1[Event] --> Bus[(Stream / Kafka Topic)]
    Bus --> SP[Stream Processor - e.g. Flink]
    SP -->|tumbling 60s window| Agg[Real-Time Aggregate]
    Agg --> Alert[Live Dashboard / Alert]
```
*Events যেভাবে আসে সেভাবে ক্রমাগত প্রসেস করা হয়, near real-time ফলাফলের জন্য ছোট time window-এ aggregate করা হয়।*

## ৩. Lambda Architecture: Batch এবং Speed Layers মিলিত

```mermaid
flowchart TB
    Source[Incoming Data] --> Batch[Batch Layer - accurate, slow]
    Source --> Speed[Speed Layer - fast, approximate]
    Batch --> Serving[Serving Layer]
    Speed --> Serving
    Serving --> Query[Query Result: accurate + fresh]
```
*Lambda architecture ডেটাকে একটা ধীর accurate batch layer এবং একটা দ্রুত approximate speed layer উভয়ের মধ্য দিয়ে চালায়, এবং query time-এ দুটো view merge করে।*
