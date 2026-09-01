# Diagrams: Testing Distributed Systems

## 1. Three Kinds of Load Testing

```mermaid
flowchart TB
    L["Load Test<br/>Traffic at expected peak"] --> LQ["Question: Does it hold up<br/>at the traffic we actually expect?"]
    S["Stress Test<br/>Traffic well beyond expected peak"] --> SQ["Question: Where's the breaking point,<br/>and does it fail gracefully or catastrophically?"]
    E["Soak Test<br/>Moderate load, sustained for hours/days"] --> EQ["Question: Does it degrade slowly over time<br/>(leaks, resource exhaustion)?"]
```
*Each testing type answers a distinct question about how the system behaves under traffic — expected, extreme, or sustained.*

## 2. The Chaos Engineering Loop

```mermaid
flowchart LR
    H["1. Form a hypothesis<br/>(e.g., failover completes in 10s)"] --> B["2. Define blast radius<br/>(small % of traffic, abort trigger set)"]
    B --> R["3. Run the experiment<br/>(inject real failure)"]
    R --> O["4. Observe actual behavior<br/>vs. the hypothesis"]
    O --> F["5. Fix what didn't match"]
    F --> H2["6. Expand blast radius<br/>and try more sophisticated experiments"]
    H2 -.->|next cycle| H
```
*Chaos engineering is a disciplined, repeatable loop — not random destruction — that almost always surfaces a gap between designed and actual behavior, which is exactly its value.*

## 3. Chaos Experiment vs. Game Day: What Each Actually Tests

```mermaid
flowchart TB
    subgraph Chaos["Chaos Experiment"]
        C1[Kill primary DB replica] --> C2["Automated failover triggers<br/>Tests: does the SYSTEM recover correctly?"]
    end

    subgraph GameDay["Game Day"]
        G1[Simulate region outage] --> G2["On-call engineer investigates,<br/>follows runbook, escalates as needed"]
        G2 --> G3["Tests: do the PEOPLE and PROCESS<br/>respond correctly under pressure?"]
    end
```
*A chaos experiment tests whether the automated system recovers as designed. A game day goes further, testing whether the humans and documented procedures around it actually work when something breaks.*
