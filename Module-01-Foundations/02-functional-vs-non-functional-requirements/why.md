# Why This Topic Matters: Functional vs Non-Functional Requirements

> **In one sentence:** Functional requirements tell you *what to build*; non-functional requirements tell you *how it must behave* — and almost every architectural decision is driven by the second kind, which is exactly the kind teams forget to write down.

## The World Before This Idea

A product spec says: "Users can upload a profile photo." The team builds it. It works. Then:

- Someone uploads a 400 MB TIFF and the server runs out of memory.
- The upload takes 45 seconds on mobile and users abandon the flow.
- Legal asks where the photos are stored, because EU users' data can't leave the EU.
- The photo service goes down and takes the whole login page with it.

Every one of those is a requirement that was real from day one but was never stated. The feature was "done" and the system was still wrong.

## The Problems It Solves

### 1. Building the right feature with the wrong properties
**What you see:** The functionality is exactly as specified, and it's still unusable — too slow, too fragile, or too expensive to run.

**Why it happens:** Specs are written in terms of user-visible behavior, which is the functional half. Latency, throughput, availability, durability, and cost are invisible in a spec but decisive in an architecture. A "send message" feature has a completely different design if it must deliver in under 100 ms versus "eventually."

**How this distinction solves it:** It makes you ask, for every feature, a second set of questions: how fast, how many, how available, how durable, how secure. Those answers — not the feature list — determine whether you need a queue, a cache, a replica, or a CDN.

### 2. Architecture arguments with no tiebreaker
**What you see:** "Should we use Postgres or DynamoDB?" turns into a religious debate. Both sides have good points and nobody can close it.

**Why it happens:** Without stated non-functional targets, every option is defensible. Technology choices are only decidable *relative to constraints*.

**How this distinction solves it:** Write down "99.99% availability, 50k writes/sec, strong consistency on balance updates" and the option space collapses fast. Requirements are what turn architecture from taste into engineering.

### 3. Scope that quietly triples
**What you see:** A two-week feature ships in three months because "we also needed" audit logging, rate limiting, retries, and an admin tool.

**Why it happens:** Cross-cutting non-functional needs (security, observability, compliance, operability) are discovered one at a time, mid-build, when each one is maximally expensive to retrofit.

**How this distinction solves it:** Surfacing them as first-class requirements at the start means they're either budgeted, deferred deliberately, or dropped deliberately — instead of ambushing the delivery date.

### 4. Interviews where you design the wrong system
**What you see:** A candidate is asked to design a URL shortener and spends twenty minutes on the hashing algorithm, never asking how many URLs, what read/write ratio, or whether links expire.

**Why it happens:** The prompt is deliberately vague. Interviewers omit the constraints to see whether you'll ask for them.

**How this distinction solves it:** Opening with requirement clarification — functional first, then scale/latency/availability/consistency — is the single highest-scoring move in a design interview, because it demonstrates you know that the constraints *are* the problem.

## The Price You Pay

- **Requirement theater.** Long documents full of "the system shall be highly available" with no number attached are worse than nothing; they create false confidence.
- **Over-specification early.** Committing to 99.999% availability before you have a single user forces expensive architecture for a load that may never arrive.
- **Time cost.** Genuinely eliciting non-functional requirements means talking to product, legal, and ops — slower than just coding, and often worth it.

## When You Need It — and When You Don't

| Nail down requirements when | Move fast without when |
|---|---|
| The system handles money, identity, or regulated data | It's a throwaway prototype or spike |
| You're choosing infrastructure that's costly to swap | You're adding a button to an existing page |
| Multiple teams will integrate against it | The blast radius is a single internal tool |
| You're in a system design interview — *always* | — |

## Why This Shows Up in Interviews

It's the opening move of every system design round, and it's scored heavily. A candidate who says "Before I design, let me clarify: how many daily active users, what's the read-to-write ratio, do we need strong consistency here, and what's the latency target?" has already demonstrated more seniority than one who starts drawing boxes. Everything drawn later should trace back to one of those answers.

## How It Connects

Non-functional requirements are the reason the rest of this course exists. *Scalability* answers "how many," *availability and fault tolerance* answer "how reliable," *caching and CDNs* answer "how fast," *consistency models* answer "how correct." Each later topic is a tool for satisfying a class of non-functional requirement — so learning to name the requirement is learning which tool to reach for.

**Next:** [Client-Server Architecture & How the Internet Works](../03-client-server-architecture-and-how-the-internet-works/why.md) — the substrate every requirement ultimately runs on.
