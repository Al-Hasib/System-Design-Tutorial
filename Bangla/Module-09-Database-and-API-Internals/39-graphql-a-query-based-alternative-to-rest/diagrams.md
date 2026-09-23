# Diagrams: GraphQL vs. REST

## ১. REST-এ Under-Fetching বনাম একটি GraphQL Query

```mermaid
flowchart TB
    subgraph REST["REST: 3 Round Trips"]
        C1[Client] --> R1["GET /users/5"]
        C1 --> R2["GET /users/5/orders?limit=3"]
        C1 --> R3["GET /users/5/notifications?limit=5"]
    end

    subgraph GQL["GraphQL: 1 Round Trip"]
        C2[Client] --> Q["POST /graphql<br/>query specifies user + orders + notifications"]
        Q --> Resolve[Server resolves all three fields<br/>and returns one combined response]
    end
```

*একটি screen-এর জন্য একাধিক resource থেকে data assemble করতে একাধিক REST round trip খরচ হয়, অথবা একটি GraphQL query যা পুরো combined shape আগে থেকেই নির্দিষ্ট করে দেয়।*

## ২. N+1 Query Problem এবং Batching

```mermaid
flowchart TB
    subgraph Naive["Naive Resolvers: N+1 Queries"]
        U1[Query: 20 users + their orders] --> Q1[1 query: fetch 20 users]
        Q1 --> Q2["20 separate queries:<br/>fetch orders for user 1<br/>fetch orders for user 2<br/>... fetch orders for user 20"]
    end

    subgraph Batched["With DataLoader: Batched Queries"]
        U2[Query: 20 users + their orders] --> B1[1 query: fetch 20 users]
        B1 --> B2["1 batched query:<br/>fetch orders WHERE user_id IN (20 ids)"]
    end
```

*Naive per-field resolvers একটি list-এর প্রতিটি item-এর জন্য একটি করে query trigger করতে পারে। Batching একই data type-এর জন্য সব requests একটি query execution-এর মধ্যে সংগ্রহ করে একটি single call-এ পরিণত করে।*

## ৩. প্রতিটি API Style কোথায় মানানসই

```mermaid
flowchart LR
    Mobile[Mobile App<br/>needs subset of fields] --> GQL[GraphQL Endpoint]
    Web[Web App<br/>needs different subset] --> GQL
    TV[Smart TV App<br/>needs yet another subset] --> GQL
    GQL --> Schema[(Shared GraphQL Schema<br/>+ Resolvers)]

    Partner[Third-Party Partner<br/>needs simple, cacheable access] --> REST[REST API]
    REST --> Schema2[(Fixed-Shape Resources)]
```

*ভিন্ন, evolving data প্রয়োজনসহ একাধিক client type কার্যকরভাবে একটি GraphQL schema share করে; সহজ, cacheable, well-documented access প্রয়োজন এমন একটি একক external partner-কে প্রায়ই এখনও REST দ্বারা ভালোভাবে সার্ভ করা হয়।*
