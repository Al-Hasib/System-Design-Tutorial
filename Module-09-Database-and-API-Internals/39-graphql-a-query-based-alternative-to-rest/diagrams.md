# Diagrams: GraphQL vs. REST

## 1. Under-Fetching in REST vs. One GraphQL Query

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
*Assembling data from multiple resources for one screen costs multiple REST round trips, or one GraphQL query that specifies the whole combined shape upfront.*

## 2. The N+1 Query Problem and Batching

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
*Naive per-field resolvers can trigger one query per item in a list. Batching collects all requests for the same data type within one query execution into a single call.*

## 3. Where Each API Style Fits

```mermaid
flowchart LR
    Mobile[Mobile App<br/>needs subset of fields] --> GQL[GraphQL Endpoint]
    Web[Web App<br/>needs different subset] --> GQL
    TV[Smart TV App<br/>needs yet another subset] --> GQL
    GQL --> Schema[(Shared GraphQL Schema<br/>+ Resolvers)]

    Partner[Third-Party Partner<br/>needs simple, cacheable access] --> REST[REST API]
    REST --> Schema2[(Fixed-Shape Resources)]
```
*Multiple client types with different, evolving data needs share one GraphQL schema efficiently; a single external partner needing simple, cacheable, well-documented access is often still better served by REST.*
