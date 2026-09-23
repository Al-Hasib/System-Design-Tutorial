# Diagrams: Forward Proxy vs Reverse Proxy

## ১. Forward Proxy — Client-কে Protect করা

```mermaid
flowchart LR
    C1[Employee Laptop A] --> FP[Forward Proxy]
    C2[Employee Laptop B] --> FP
    FP --> Internet((Internet))
    Internet --> Server[Destination Website]

    Server -.->|"sees only the proxy's IP"| FP
```

*Destination server শুধুমাত্র forward proxy-র identity দেখতে পায় — কোন নির্দিষ্ট client request করেছে তার কোনো visibility এটির নেই।*

## ২. Reverse Proxy — Server-কে Protect করা

```mermaid
flowchart LR
    Client[Customer Browser] --> RP[Reverse Proxy]
    RP --> S1[Backend Server 1]
    RP --> S2[Backend Server 2]
    RP --> S3[Backend Server 3]

    Client -.->|"sees only the reverse proxy"| RP
```

*Client বিশ্বাস করে যে সে সরাসরি "the server"-এর সাথে কথা বলছে — এটি কখনো জানতে পারে না যে reverse proxy-র পেছনে backend server-এর একটি fleet বিদ্যমান।*

## ৩. Side-by-Side: একই Traffic Path, বিপরীত Purpose

```mermaid
flowchart TB
    subgraph Forward["Forward Proxy Flow"]
        direction LR
        FC[Client] --> FProxy[Forward Proxy] --> FS[Server]
    end
    subgraph Reverse["Reverse Proxy Flow"]
        direction LR
        RC[Client] --> RProxy[Reverse Proxy] --> RS[Backend Servers]
    end
```

*Structurally একই — client, proxy, server — কিন্তু forward proxy client-পক্ষ দ্বারা deploy করা হয় এবং তাদের represent করে, যেখানে reverse proxy server-পক্ষ দ্বারা deploy করা হয় এবং তাদের represent করে।*
