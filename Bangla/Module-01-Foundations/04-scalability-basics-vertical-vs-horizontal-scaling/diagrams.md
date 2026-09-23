# Diagrams: Scalability Basics — Vertical vs Horizontal Scaling

## ১. Vertical Scaling

```mermaid
flowchart LR
    A[Server\n2 CPU / 4GB RAM] -->|Upgrade| B[Same Server\n16 CPU / 64GB RAM]
```
*Caption: Vertical scaling একটি machine-কে তার নিজের একটি আরও শক্তিশালী সংস্করণ দিয়ে প্রতিস্থাপন করে — একই single point of failure রয়ে যায়।*

## ২. Horizontal Scaling

```mermaid
flowchart TD
    LB[Load Balancer] --> S1[Server 1]
    LB --> S2[Server 2]
    LB --> S3[Server 3]
    LB --> S4[Server 4]
    S1 --> DB[(Shared Database)]
    S2 --> DB
    S3 --> DB
    S4 --> DB
```
*Caption: Horizontal scaling একটি load balancer-এর পেছনে আরও machine যোগ করে, সবগুলো একটি common data store শেয়ার করে।*

## ৩. Growth Path: Vertical First, Then Horizontal

```mermaid
flowchart LR
    A[Small App\nSingle Small Server] -->|Traffic grows| B[Vertical Scaling\nBigger Single Server]
    B -->|Hits ceiling / needs redundancy| C[Horizontal Scaling\nMultiple Servers + Load Balancer]
```
*Caption: একটি সাধারণ scaling যাত্রা — সরলতার জন্য vertical scaling দিয়ে শুরু করুন, বৃদ্ধির চাহিদা অনুযায়ী horizontal scaling-এ স্থানান্তরিত হন।*
