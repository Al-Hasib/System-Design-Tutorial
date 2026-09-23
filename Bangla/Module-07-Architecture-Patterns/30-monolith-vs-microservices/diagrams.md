# ডায়াগ্রাম: Monolith vs Microservices

## ১. Monolithic Architecture

```mermaid
flowchart TB
    Client[Client Apps] --> LB[Load Balancer]
    LB --> M1[Monolith Instance 1]
    LB --> M2[Monolith Instance 2]

    subgraph "Monolith Process"
        direction TB
        UI[Presentation Layer]
        BL[Business Logic: Users, Orders, Payments, Notifications]
        DAL[Data Access Layer]
        UI --> BL --> DAL
    end

    M1 -.-> UI
    M2 -.-> UI
    DAL --> DB[(Single Shared Database)]
```
*একটি monolith সব module-কে একটি একক unit হিসেবে deploy করে, একটি load balancer-এর পেছনে replicate করা হয়, এবং একটি database share করে।*

## ২. Microservices Architecture

```mermaid
flowchart LR
    Client[Client Apps] --> GW[API Gateway]
    GW --> UserSvc[User Service]
    GW --> OrderSvc[Order Service]
    GW --> PaySvc[Payment Service]
    GW --> NotifSvc[Notification Service]

    UserSvc --> UserDB[(User DB)]
    OrderSvc --> OrderDB[(Order DB)]
    PaySvc --> PayDB[(Payment DB)]

    OrderSvc -- async event --> Queue[[Message Queue]]
    Queue --> NotifSvc
```
*প্রতিটি microservice স্বাধীনভাবে deployable, নিজস্ব database-এর মালিক, এবং network-এর মাধ্যমে সরাসরি বা একটি message queue-র মাধ্যমে যোগাযোগ করে।*

## ৩. Strangler Fig Migration Path

```mermaid
flowchart LR
    Client[Client Apps] --> GW[API Gateway / Facade]
    GW -- legacy routes --> Mono[Remaining Monolith]
    GW -- new route: /payments --> PaySvc[New Payment Service]
    GW -- new route: /notifications --> NotifSvc[New Notification Service]
    Mono --> DB[(Legacy Database)]
    PaySvc --> PayDB[(Payment DB)]
    NotifSvc --> NotifDB[(Notification DB)]
```
*Strangler fig pattern নতুন-বের-করা capability-গুলোর জন্য traffic নতুন service-এ route করে, যখন সংকুচিত monolith বাকি সবকিছু সেবা দিতে থাকে।*
</content>
