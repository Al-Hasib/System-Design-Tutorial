# ডায়াগ্রাম: Availability, Reliability, Redundancy & Fault Tolerance

## ১. Single Point of Failure বনাম Redundant ডিজাইন

```mermaid
flowchart TD
    subgraph SPOF["No Redundancy (Single Point of Failure)"]
        C1[Client] --> S1[Single Server] --> D1[(Single Database)]
    end
```
*Caption: একটি একক server এবং একক database মানে যেকোনো একটি ব্যর্থতা পুরো সিস্টেমকে ধ্বসিয়ে দেয়।*

```mermaid
flowchart TD
    subgraph Redundant["Redundant Design"]
        C2[Client] --> LB[Load Balancer]
        LB --> S2[Server A]
        LB --> S3[Server B]
        S2 --> D2[(Primary DB)]
        S3 --> D2
        D2 -.replication.-> D3[(Replica DB)]
    end
```
*Caption: Redundant server এবং একটি replicated database মানে কোনো একক ব্যর্থতা সিস্টেমকে ধ্বসিয়ে দেয় না।*

## ২. Health Check-এর মাধ্যমে Failover ক্রম

```mermaid
sequenceDiagram
    participant LB as Load Balancer
    participant S1 as Server A (healthy)
    participant S2 as Server B (fails)
    LB->>S2: Health check
    S2--xLB: No response (down)
    LB->>S1: Route traffic here instead
    Note over LB,S2: Server B removed from rotation until healthy again
```
*Caption: Health check একটি load balancer-কে একটি ব্যর্থ server শনাক্ত করতে এবং স্বয়ংক্রিয়ভাবে একটি সুস্থ server-এ ট্র্যাফিক failover করতে দেয়।*

## ৩. The Nines: Downtime তুলনা

```mermaid
flowchart LR
    A["99%\n~3.65 days/year"] --> B["99.9%\n~8.76 hours/year"]
    B --> C["99.99%\n~52.6 minutes/year"]
    C --> D["99.999%\n~5.26 minutes/year"]
```
*Caption: Availability-র প্রতিটি অতিরিক্ত "nine" বার্ষিক অনুমোদিত downtime-এ প্রায় 10x হ্রাস নির্দেশ করে।*
</content>
