# Diagrams: Testing Distributed Systems

## ১. তিন ধরনের Load Testing

```mermaid
flowchart TB
    L["Load Test<br/>Traffic at expected peak"] --> LQ["Question: Does it hold up<br/>at the traffic we actually expect?"]
    S["Stress Test<br/>Traffic well beyond expected peak"] --> SQ["Question: Where's the breaking point,<br/>and does it fail gracefully or catastrophically?"]
    E["Soak Test<br/>Moderate load, sustained for hours/days"] --> EQ["Question: Does it degrade slowly over time<br/>(leaks, resource exhaustion)?"]
```
*প্রতিটা testing type সিস্টেম traffic-এর অধীনে—প্রত্যাশিত, extreme, বা sustained—কীভাবে আচরণ করে তা নিয়ে একটা আলাদা প্রশ্নের উত্তর দেয়।*

## ২. Chaos Engineering Loop

```mermaid
flowchart LR
    H["1. Form a hypothesis<br/>(e.g., failover completes in 10s)"] --> B["2. Define blast radius<br/>(small % of traffic, abort trigger set)"]
    B --> R["3. Run the experiment<br/>(inject real failure)"]
    R --> O["4. Observe actual behavior<br/>vs. the hypothesis"]
    O --> F["5. Fix what didn't match"]
    F --> H2["6. Expand blast radius<br/>and try more sophisticated experiments"]
    H2 -.->|next cycle| H
```
*Chaos engineering একটা disciplined, repeatable loop—random ধ্বংস নয়—যা প্রায় সবসময়ই ডিজাইন করা আচরণ ও প্রকৃত আচরণের মধ্যে একটা ফাঁক সামনে আনে, যা ঠিক এর মূল্য।*

## ৩. Chaos Experiment বনাম Game Day: প্রতিটা আসলে কী টেস্ট করে

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
*একটা chaos experiment টেস্ট করে automated system ডিজাইন অনুযায়ী recover করে কিনা। একটা game day আরও এগিয়ে গিয়ে টেস্ট করে এর চারপাশের মানুষ এবং documented procedure কিছু ভেঙে গেলে আসলেই কাজ করে কিনা।*
