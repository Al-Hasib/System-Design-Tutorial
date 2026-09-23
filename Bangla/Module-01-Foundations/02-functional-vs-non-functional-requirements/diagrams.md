# Diagrams: Functional vs Non-Functional Requirements

## ১. Requirements-এর বিভাজন

```mermaid
flowchart TD
    R[Gather Requirements] --> F["Functional Requirements<br/>'What it does'"]
    R --> N["Non-Functional Requirements<br/>'How well it does it'"]
    F --> F1[Upload photo]
    F --> F2[Follow user]
    F --> F3[View feed]
    N --> N1["Latency &lt; 300ms"]
    N --> N2[99.99% availability]
    N --> N3[500M DAU scalability]
```

*Caption: প্রতিটি design শুরু হয় requirements-কে দুই ভাগে বিভক্ত করে — system কী করে বনাম এটা কতটা ভালোভাবে করতে হয়।*

## ২. একটি Interview-এ Requirements Gathering-এর Flow

```mermaid
sequenceDiagram
    participant Interviewer
    participant Candidate
    Interviewer->>Candidate: Design Instagram
    Candidate->>Interviewer: How many daily active users?
    Interviewer->>Candidate: ~500 million
    Candidate->>Interviewer: Read/write ratio? Consistency needs?
    Interviewer->>Candidate: Read-heavy, eventual consistency OK
    Candidate->>Interviewer: Great, I'll design around those constraints
```

*Caption: কোনো architecture প্রস্তাব করার আগে, clarifying questions একটি অস্পষ্ট প্রম্পটকে concrete, design-চালিত requirements-এ রূপান্তরিত করে।*

## ৩. Back-of-the-Envelope Estimation Pipeline

```mermaid
flowchart LR
    DAU[Daily Active Users] --> Req[Requests per Day]
    Req --> AvgRPS[Average RPS\n÷ 86,400 sec]
    AvgRPS --> PeakRPS[Peak RPS\n× 2-3x]
    PeakRPS --> Design[Architecture Decisions\ne.g. need load balancing?]
```

*Caption: গুণ ও ভাগের একটি সাধারণ শৃঙ্খল "অনেক users"-কে এমন concrete সংখ্যায় রূপান্তরিত করে যা প্রকৃত design সিদ্ধান্তকে চালিত করে।*
</content>
