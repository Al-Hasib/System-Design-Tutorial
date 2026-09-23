# ডায়াগ্রাম: Event-Driven Architecture

## ১. Request-Driven বনাম Event-Driven Control Flow

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
*Request-driven flow একটি সরাসরি call-এ block হয়ে থাকে; event-driven flow যেকোনো সংখ্যক services-কে স্বাধীনভাবে এবং asynchronously প্রতিক্রিয়া জানাতে দেয়।*

## ২. Event-Driven প্রতিক্রিয়া Chain (Netflix-ধাঁচের উদাহরণ)

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
*একটি একক "playback completed" event তিনটি স্বাধীন, decoupled প্রতিক্রিয়া trigger করে।*

## ৩. Event Sourcing: একটি Event Log থেকে উদ্ভূত State

```mermaid
flowchart LR
    E1[AccountOpened] --> E2[Deposit +100]
    E2 --> E3[Withdrawal -30]
    E3 --> E4[Deposit +50]
    E4 --> S["Current Balance = 120 (replayed from log)"]
```
*Event sourcing-এ, বর্তমান state সরাসরি সংরক্ষণ করার বদলে events-এর সম্পূর্ণ history replay করে গণনা করা হয়।*
</content>
