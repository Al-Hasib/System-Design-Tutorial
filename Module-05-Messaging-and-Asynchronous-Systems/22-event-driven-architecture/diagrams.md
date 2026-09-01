# Diagrams: Event-Driven Architecture

## 1. Request-Driven vs Event-Driven Control Flow

```mermaid
flowchart TB
    subgraph RD[Request-Driven]
        direction LR
        A1[Service A] -->|call, wait for response| B1[Service B]
    end
    subgraph ED[Event-Driven]
        direction LR
        A2[Service A] -->|publish event| Bus[(Event Bus)]
        Bus --> C1[Service B reacts]
        Bus --> C2[Service C reacts]
        Bus --> C3[Service D reacts]
    end
```
*Request-driven flow blocks on a direct call; event-driven flow lets any number of services react independently and asynchronously.*

## 2. Event-Driven Reaction Chain (Netflix-style Example)

```mermaid
sequenceDiagram
    participant Player as Playback Service
    participant Bus as Event Bus
    participant Rec as Recommendation Engine
    participant CW as Continue Watching
    participant Bill as Usage/Billing Analytics

    Player->>Bus: publish PlaybackCompleted event
    Bus->>Rec: deliver event
    Bus->>CW: deliver event
    Bus->>Bill: deliver event
    Note over Rec,Bill: Each service reacts independently, on its own schedule
```
*A single "playback completed" event triggers three independent, decoupled reactions.*

## 3. Event Sourcing: State Derived from an Event Log

```mermaid
flowchart LR
    E1[AccountOpened] --> E2[Deposit +100]
    E2 --> E3[Withdrawal -30]
    E3 --> E4[Deposit +50]
    E4 --> S["Current Balance = 120 (replayed from log)"]
```
*In event sourcing, the current state is computed by replaying the full history of events rather than being stored directly.*
