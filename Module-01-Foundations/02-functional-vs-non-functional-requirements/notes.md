# Notes: Functional vs Non-Functional Requirements

## Definitions

| Type | Definition | Answers | Example |
|---|---|---|---|
| **Functional Requirement** | What the system should do — features and behaviors | "What can a user do?" | Upload a photo, follow a user, like a post |
| **Non-Functional Requirement** | How well the system must perform those functions | "How fast/reliable/secure must it be?" | 99.99% uptime, <300ms latency, 500M DAU |

## Common Non-Functional Categories

| Category | Question it Answers |
|---|---|
| Performance/Latency | How fast must responses be? |
| Scalability | How much traffic/data/users must it handle, now and later? |
| Availability | What % uptime is required? |
| Consistency | How fast must data changes propagate/be visible everywhere? |
| Durability | Can confirmed data ever be lost? |
| Security | What must be protected, from whom? |
| Cost | What's the infrastructure budget? |

## Back-of-the-Envelope Estimation Recipe

1. Start with **Daily Active Users (DAU)**.
2. Multiply by **actions per user per day** → total daily requests.
3. Divide by **86,400 seconds/day** → average requests per second (RPS).
4. Multiply average RPS by **2-3x** → peak RPS (traffic is not evenly distributed).
5. For storage: **items/day × avg size per item** → daily storage growth.

### Example Calculation

- 100M DAU × 10 requests/day = 1B requests/day
- 1B / 86,400 ≈ 11,600 RPS average
- 11,600 × 2.5 ≈ ~29,000 RPS peak
- 10M photo uploads/day × 2MB = 20TB/day new storage

## Key Contrast Example

| System | Functional Req | Dominant Non-Functional Req |
|---|---|---|
| URL Shortener | Shorten URL, redirect | Ultra-low read latency, huge read:write ratio |
| Banking System | Transfer money | Strong consistency, zero data loss (durability) |

## Quick Revision Bullets

- Functional = "what"; Non-functional = "how well."
- Always ask clarifying questions before designing (scale, read/write ratio, consistency needs, global vs regional).
- Same feature set + different non-functional requirements = different architecture.
- Rough math (not exact numbers) is the goal of capacity estimation.
