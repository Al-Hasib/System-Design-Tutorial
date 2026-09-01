# Diagrams: Consensus Algorithms (Paxos & Raft)

## 1. Paxos — Prepare/Promise and Accept/Accepted Phases

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

Caption: A value is chosen once a majority of Acceptors accept the same proposal number and value — the Proposer never needs all N Acceptors, only a quorum.

## 2. Raft — Node State Transitions

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

Caption: Every Raft node is a Follower, Candidate, or Leader; a randomized election timeout drives Follower to Candidate, and only a majority vote promotes a Candidate to Leader for that term.

## 3. Raft — Log Replication

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

Caption: The Leader commits a log entry as soon as a majority of the cluster (including itself) has durably appended it, then informs Followers of the new commit index on the next heartbeat.
