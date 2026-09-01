# Multi-Region Architecture & Disaster Recovery

**Difficulty:** Advanced
**Estimated length:** 16-20 min
**Prerequisites:** [05 - Availability, Reliability & Fault Tolerance](../../Module-01-Foundations/05-availability-reliability-and-fault-tolerance/README.md), [15 - CAP Theorem & PACELC](../../Module-03-Databases-and-Storage/15-cap-theorem-and-pacelc/README.md)

## Learning Objectives

- Explain why redundancy within a single region isn't enough protection against every real failure scenario.
- Compare active-passive and active-active multi-region architectures and their trade-offs.
- Define RTO and RPO and explain why they're business decisions, not purely technical ones.
- Explain the specific challenges multi-region write availability creates, connecting back to CAP/PACELC.
- Design an appropriate disaster recovery strategy for a system given its actual availability and consistency requirements.

## Script

### Hook / Intro

Back in Module 1, we covered redundancy: multiple servers, multiple database replicas, so a single machine failing doesn't take your system down. That's real protection — against a single machine failing. It does nothing if the entire facility that machine lives in loses power, catches fire, or becomes unreachable due to a networking failure at the data center or cloud region level — a rarer event, but not a hypothetical one; every major cloud provider has had region-level outages. Today we cover the next level up: multi-region architecture, and the disaster recovery planning that defines exactly how much data loss and downtime your business is actually willing to accept when the unlikely happens.

### Why Redundancy Within a Region Isn't Enough

A "region" in cloud infrastructure terms is a geographically distinct location, physically isolated from other regions specifically so a failure in one (power, networking, a natural disaster, human error affecting shared regional infrastructure) doesn't propagate to another. Redundancy within a region — multiple availability zones, multiple servers, replicated databases — protects against failures at the machine or rack level, and even at the level of one data center within a region. It does not protect against something affecting the entire region simultaneously. Whether multi-region protection is worth its cost depends entirely on the business: a personal blog can tolerate a few hours of regional downtime; a payment processor, a healthcare system, or global e-commerce platform generally cannot.

### Active-Passive: A Standby Region

The simpler multi-region pattern is **active-passive** (also called active-standby): one region (the "active" or "primary") serves all live traffic, while a second region (the "passive" or "standby") sits ready with a replicated copy of the data, but doesn't serve production traffic under normal operation. If the active region fails, you **fail over** — redirect traffic to the passive region, which now becomes active. This is conceptually simple and avoids the hardest problems of multi-region writes (more on that shortly), but it has two real costs: the standby region's capacity sits mostly idle, paid for but unused most of the time, and failover itself takes time — DNS propagation, application warm-up, verifying data is current — which is exactly why disaster recovery planning defines specific target numbers for that gap rather than leaving it as "however long it takes."

### Active-Active: Every Region Serves Traffic

**Active-active** goes further: multiple regions simultaneously serve live production traffic, typically routed to the geographically nearest region for lower latency (echoing the CDN/anycast routing ideas from Module 4). If one region fails, the others simply absorb its traffic — no failover delay, because there was never a single active region to fail over from in the first place, and the idle-standby-capacity cost mostly disappears since every region's capacity is doing real work all the time. The cost shows up somewhere else entirely: **data consistency**. If users in two different regions can both write to what's supposed to be the same logical data at the same time, you're back to the CAP theorem and PACELC trade-offs from Module 3 — do you enforce strong consistency across regions (accepting the latency cost of cross-region coordination on every write, and reduced availability during a network partition between regions) or accept eventual consistency (fast, always-available local writes, but needing the same conflict-detection tools — recall vector clocks from earlier this module — to resolve genuinely concurrent, conflicting writes across regions). There's no version of active-active that avoids this trade-off; it's the direct, unavoidable cost of writes being accepted in more than one place.

### RTO and RPO: The Numbers That Define "How Bad"

Disaster recovery planning is formalized around two specific metrics, and getting them right is fundamentally a business decision, not a purely technical one. **RTO (Recovery Time Objective)** is the maximum acceptable time between a disaster occurring and the system being back up and serving traffic again — "we must be back online within 15 minutes." **RPO (Recovery Point Objective)** is the maximum acceptable amount of data loss, measured in time — "we can afford to lose at most 5 minutes of the most recent writes." These two numbers directly drive architecture decisions: an RPO of zero (no data loss whatsoever is acceptable) demands synchronous cross-region replication on every write, paying real latency on every single write in exchange for that guarantee; an RPO of a few minutes can tolerate asynchronous replication, which is faster and cheaper day-to-day, accepting that a handful of the very latest writes might not have replicated yet at the exact moment of a disaster. Similarly, an RTO of seconds effectively requires active-active (no failover delay at all); an RTO of hours can be met with active-passive, or in the least demanding cases, even a documented manual recovery process from backups. Neither number has a universally "correct" answer — they're set by weighing the cost of stronger guarantees against the actual business cost of downtime or data loss for this specific system.

### Real-World Example

Consider a company running an internal analytics dashboard (low criticality) versus its customer-facing payment processing system (high criticality) in the same cloud provider. The analytics dashboard might reasonably have an RTO of "a few hours" and an RPO of "up to a day" — a documented, mostly-manual recovery-from-backup process is genuinely fine here, because the business cost of a slow, lossy recovery for an internal tool is low. The payment system might have an RTO of "under a minute" and an RPO of "zero data loss" — demanding active-active architecture with synchronous cross-region replication for transaction data specifically, even though that adds real latency to every payment write, because the business cost of losing even one confirmed transaction, or being down for more than a minute during business hours, is unacceptably high. Same company, same cloud provider, two entirely different, deliberately chosen architectures — driven by two different sets of RTO/RPO numbers that someone in the business, not just engineering, explicitly signed off on.

### Recap

Redundancy within a single region protects against machine and data-center-level failures, but not against an entire region going down — a rarer but real risk that multi-region architecture exists to address. Active-passive keeps a standby region ready and fails over when needed, simpler but with idle capacity cost and a failover delay. Active-active serves traffic from every region simultaneously, eliminating failover delay and idle capacity at the cost of confronting cross-region write consistency trade-offs head-on (the CAP/PACELC trade-off, unavoidable once writes happen in more than one place). RTO and RPO are the specific, business-driven numbers — maximum acceptable downtime and maximum acceptable data loss — that determine which of these architectures (and which replication strategy) a given system actually needs, and they should be decided deliberately, not left implicit.

### What's Next

That closes out this module on observability, deployment, and production operations — and with it, the deep-dive content of this course. From here, everything comes together in Module 12's full, end-to-end case studies, starting with a classic warm-up: designing a URL shortener from a blank whiteboard.

## Key Takeaways

- Redundancy within one region protects against machine/data-center failures but not region-level failures — multi-region architecture exists specifically for that rarer, higher-impact risk.
- Active-passive keeps a standby region ready, failing over when the active region fails — simpler, but with idle standby capacity cost and a real failover delay.
- Active-active serves traffic from every region simultaneously, eliminating failover delay, but forces an explicit CAP/PACELC trade-off on cross-region writes since data can be written in more than one place at once.
- RTO (max acceptable downtime) and RPO (max acceptable data loss) are business-driven numbers that directly determine the right architecture and replication strategy — not universal constants.
- The right multi-region/DR strategy differs by system even within the same company — driven by each system's actual business cost of downtime and data loss, not a one-size-fits-all default.
