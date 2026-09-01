# Diagrams: Functional vs Non-Functional Requirements

## 1. Requirements Split

```mermaid
flowchart TD
    R[Gather Requirements] --> F[Functional Requirements\n\"What it does\"]
    R --> N[Non-Functional Requirements\n\"How well it does it\"]
    F --> F1[Upload photo]
    F --> F2[Follow user]
    F --> F3[View feed]
    N --> N1[Latency < 300ms]
    N --> N2[99.99% availability]
    N --> N3[500M DAU scalability]
```
*Caption: Every design starts by splitting requirements into what the system does versus how well it must do it.*

## 2. Requirements Gathering Flow in an Interview

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
*Caption: Clarifying questions turn a vague prompt into concrete, design-driving requirements before any architecture is proposed.*

## 3. Back-of-the-Envelope Estimation Pipeline

```mermaid
flowchart LR
    DAU[Daily Active Users] --> Req[Requests per Day]
    Req --> AvgRPS[Average RPS\n÷ 86,400 sec]
    AvgRPS --> PeakRPS[Peak RPS\n× 2-3x]
    PeakRPS --> Design[Architecture Decisions\ne.g. need load balancing?]
```
*Caption: A simple chain of multiplication and division turns "lots of users" into concrete numbers that drive real design decisions.*
