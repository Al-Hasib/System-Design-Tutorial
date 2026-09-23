# Diagrams: Web Server Internals

## ১. Web Server-এর মূল Loop

```mermaid
flowchart LR
    A[Accept Connection] --> B[Read & Parse Request]
    B --> C["Process<br/>(business logic, DB calls, etc.)"]
    C --> D[Write Response]
    D --> A
```
*প্রতিটি web server প্রতিটি request-এর জন্য এই loop পুনরাবৃত্তি করে — performance এবং scalability সম্পূর্ণভাবে নির্ভর করে অনেক request-এর জন্য concurrently ধাপ ৩ ("Process") কীভাবে হ্যান্ডেল করা হয় তার উপর।*

## ২. Thread-per-Request বনাম Event Loop

```mermaid
flowchart TB
    subgraph TPR["Thread-per-Request Model"]
        R1[Request 1] --> T1[Dedicated Thread 1\nBLOCKS waiting on DB]
        R2[Request 2] --> T2[Dedicated Thread 2\nBLOCKS waiting on DB]
        R3[Request 3] --> T3[Dedicated Thread 3\nBLOCKS waiting on DB]
    end

    subgraph EL["Event Loop Model"]
        R4[Request 1] --> Q[Event Queue]
        R5[Request 2] --> Q
        R6[Request 3] --> Q
        Q --> Loop[Single Event Loop\nnever blocks, dispatches callbacks\nwhen I/O completes]
    end
```
*Thread-per-request একটি blocking DB call-এর পুরো সময়কাল জুড়ে, যতক্ষণই লাগুক না কেন, একটি সম্পূর্ণ OS thread আটকে রাখে। একটি event loop I/O জারি করে এগিয়ে যায়, response প্রস্তুত হলেই শুধু সেই request-এ ফিরে আসে — যা একটি thread-কে হাজার হাজার "অপেক্ষমাণ" request সার্ভ করতে দেয়।*

## ৩. Static বনাম Dynamic Request কোথায় হ্যান্ডেল হয়

```mermaid
flowchart LR
    Client[Client] --> CDN[CDN / Edge Cache]
    CDN -->|Static asset: cache HIT| Client
    CDN -->|Static asset: cache MISS, or dynamic request| Proxy[Reverse Proxy]
    Proxy -->|Static, not cached yet| Origin[Origin Static Store]
    Proxy -->|Dynamic API call| App[Application Server\nEvent Loop / Thread Pool]
    App --> DB[(Database)]
```
*Static content যত তাড়াতাড়ি সম্ভব ধরে ফেলা হয় — আদর্শভাবে CDN-এ — যাতে এটি কখনো application server-এর concurrency model স্পর্শ না করে। শুধুমাত্র প্রকৃত dynamic request গুলোই app server এবং তার database পর্যন্ত পৌঁছায়।*
