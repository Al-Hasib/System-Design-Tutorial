# Diagrams: What is System Design?

## 1. From Code to Architecture

```mermaid
flowchart LR
    A[Single Function\n\"Write correct code\"] --> B[Single Service\n\"Combine functions into an app\"]
    B --> C[Full System\n\"Clients, servers, DB, cache, queue working together\"]
    C --> D[System Design\n\"How do all pieces fit & scale?\"]
```
*Caption: System design zooms out from individual code to how entire systems of components fit together.*

## 2. Core Building Blocks of a Typical System

```mermaid
flowchart TD
    Client[Client\nBrowser / Mobile App] -->|Request| LB[Load Balancer]
    LB --> Server1[Server 1]
    LB --> Server2[Server 2]
    Server1 --> Cache[(Cache)]
    Server2 --> Cache
    Server1 --> DB[(Database)]
    Server2 --> DB
    Server1 --> Queue[[Message Queue]]
    Queue --> Worker[Background Worker]
```
*Caption: The recurring cast of characters — client, load balancer, servers, cache, database, and message queue — that this entire course will explain piece by piece.*

## 3. Course Roadmap Flow

```mermaid
flowchart LR
    M1[Module 1\nFoundations] --> M2[Module 2\nNetworking]
    M2 --> M3[Module 3\nDatabases]
    M3 --> M4[Module 4\nCaching & CDN]
    M4 --> M5[Module 5\nMessaging]
    M5 --> M6[Module 6\nDistributed Systems]
    M6 --> M7[Module 7\nArchitecture Patterns]
    M7 --> M8[Module 8\nCase Studies]
```
*Caption: Each module builds directly on the vocabulary and concepts introduced in the previous one.*
