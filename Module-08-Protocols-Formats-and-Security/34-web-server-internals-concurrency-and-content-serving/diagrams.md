# Diagrams: Web Server Internals

## 1. The Web Server's Core Loop

```mermaid
flowchart LR
    A[Accept Connection] --> B[Read & Parse Request]
    B --> C["Process<br/>(business logic, DB calls, etc.)"]
    C --> D[Write Response]
    D --> A
```
*Every web server repeats this loop for every request — performance and scalability come entirely down to how step 3 ("Process") is handled for many requests concurrently.*

## 2. Thread-per-Request vs Event Loop

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
*Thread-per-request ties up a full OS thread for the whole duration of a blocking DB call, however long that takes. An event loop dispatches the I/O and moves on, only returning to that request when the response is ready — letting one thread service thousands of "waiting" requests.*

## 3. Where Static vs Dynamic Requests Get Handled

```mermaid
flowchart LR
    Client[Client] --> CDN[CDN / Edge Cache]
    CDN -->|Static asset: cache HIT| Client
    CDN -->|Static asset: cache MISS, or dynamic request| Proxy[Reverse Proxy]
    Proxy -->|Static, not cached yet| Origin[Origin Static Store]
    Proxy -->|Dynamic API call| App[Application Server\nEvent Loop / Thread Pool]
    App --> DB[(Database)]
```
*Static content is intercepted as early as possible — ideally at the CDN — so it never touches the application server's concurrency model. Only genuinely dynamic requests reach the app server and its database.*
