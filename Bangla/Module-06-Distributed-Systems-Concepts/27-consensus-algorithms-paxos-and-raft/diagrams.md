# Diagrams: Consensus Algorithms (Paxos & Raft)

## ১. Paxos — Prepare/Promise এবং Accept/Accepted Phase

```mermaid
sequenceDiagram
    participant P as Proposer
    participant A1 as Acceptor 1
    participant A2 as Acceptor 2
    participant A3 as Acceptor 3

    Note over P,A3: Phase 1: Prepare / Promise
    P->>A1: Prepare(n=5)
    P->>A2: Prepare(n=5)
    P->>A3: Prepare(n=5)
    A1-->>P: Promise(n=5, no prior accepted value)
    A2-->>P: Promise(n=5, no prior accepted value)
    Note over P: Majority (2 of 3) promised -> proceed

    Note over P,A3: Phase 2: Accept / Accepted
    P->>A1: Accept(n=5, value=X)
    P->>A2: Accept(n=5, value=X)
    A1-->>P: Accepted(n=5, value=X)
    A2-->>P: Accepted(n=5, value=X)
    Note over P: Majority accepted value X -> X is CHOSEN
```

*Caption: একটি value chosen হয় একবার Acceptor-দের একটি majority একই proposal number এবং value accept করলে — Proposer-এর কখনো সবগুলো N Acceptor-এর প্রয়োজন হয় না, শুধু একটি quorum প্রয়োজন।*

## ২. Raft — Node State Transitions

```mermaid
stateDiagram-v2
    [*] --> Follower
    Follower --> Candidate: Election timeout elapses (no heartbeat from Leader)
    Candidate --> Candidate: Split vote / election timeout -> start new term, retry
    Candidate --> Leader: Receives votes from majority of cluster
    Candidate --> Follower: Discovers current Leader or higher term
    Leader --> Follower: Discovers a node with a higher term
    Leader --> [*]: Node crashes
    Follower --> [*]: Node crashes
```

*Caption: প্রতিটি Raft node হয় একজন Follower, Candidate, অথবা Leader; একটি randomized election timeout Follower-কে Candidate-এ পরিণত করে, এবং শুধুমাত্র একটি majority vote একজন Candidate-কে সেই term-এর জন্য Leader-এ উন্নীত করে।*

## ৩. Raft — Log Replication

```mermaid
sequenceDiagram
    participant C as Client
    participant L as Leader
    participant F1 as Follower 1
    participant F2 as Follower 2

    C->>L: Command: SET x=1
    L->>L: Append entry to local log (uncommitted)
    par Replicate to followers
        L->>F1: AppendEntries(entry x=1)
        L->>F2: AppendEntries(entry x=1)
    end
    F1-->>L: Ack
    F2-->>L: Ack
    Note over L: Majority (Leader + 1 Follower) acknowledged -> COMMIT
    L->>L: Apply entry to state machine
    L-->>C: Success
    Note over L,F2: Next heartbeat informs followers of new commit index
    L->>F1: Heartbeat (commitIndex updated)
    L->>F2: Heartbeat (commitIndex updated)
```

*Caption: Leader একটি log entry কমিট করে যত তাড়াতাড়ি cluster-এর একটি majority (নিজেকে সহ) এটি স্থায়ীভাবে যুক্ত করে ফেলেছে, তারপর পরবর্তী heartbeat-এ Follower-দের নতুন commit index জানিয়ে দেয়।*
