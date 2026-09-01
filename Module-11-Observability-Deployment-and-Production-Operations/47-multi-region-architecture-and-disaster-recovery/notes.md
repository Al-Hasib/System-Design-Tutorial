# Study Notes: Multi-Region Architecture & Disaster Recovery

## Definitions

- **Region:** A geographically distinct, physically isolated infrastructure location, so a failure in one doesn't propagate to another.
- **Active-passive (active-standby):** One region serves live traffic; a second region stays ready with replicated data but doesn't serve production traffic until failover.
- **Failover:** The process of redirecting traffic to a standby region after the active region fails.
- **Active-active:** Multiple regions simultaneously serve live production traffic, typically routed to the nearest region.
- **RTO (Recovery Time Objective):** Maximum acceptable time between a disaster and the system being back online.
- **RPO (Recovery Point Objective):** Maximum acceptable amount of data loss, measured in time (e.g., "at most 5 minutes of writes").

## Active-Passive vs. Active-Active

| Aspect | Active-Passive | Active-Active |
|---|---|---|
| Traffic serving | One region at a time | All regions simultaneously |
| Idle capacity cost | Standby region mostly unused | None — all capacity does real work |
| Failover delay | Real (DNS propagation, warm-up, verification) | None — no single active region to fail over from |
| Consistency challenge | Simpler — single write region at a time | Must confront CAP/PACELC trade-offs on cross-region writes |
| Best fit | Lower RTO tolerance is acceptable (minutes) | Very low RTO requirement, near-zero downtime |

## RTO / RPO Drive Architecture

| RPO requirement | Replication strategy |
|---|---|
| Zero data loss | Synchronous cross-region replication (real latency cost on every write) |
| A few minutes acceptable | Asynchronous replication (faster, cheaper day-to-day) |

| RTO requirement | Architecture |
|---|---|
| Seconds | Active-active (no failover delay) |
| Minutes | Active-passive with automated failover |
| Hours | Documented manual recovery from backups may be acceptable |

## Key Principle

- RTO and RPO are **business decisions**, weighing the cost of stronger guarantees (latency, infrastructure cost, engineering complexity) against the actual business cost of downtime/data loss for that specific system.
- Different systems within the same company can reasonably have very different RTO/RPO targets and architectures (e.g., internal analytics dashboard vs. customer payment processing).

## Key Numbers / Facts

- Every major cloud provider (AWS, GCP, Azure) has experienced region-level outages historically — multi-region risk is not hypothetical.
- "Zero RPO" (synchronous replication, no data loss) is achievable but adds real write latency proportional to the distance/network conditions between regions.

## Summary

- Single-region redundancy protects against machine/data-center failures, not region-level failures — multi-region architecture addresses that gap.
- Active-passive is simpler with idle-capacity cost and failover delay; active-active eliminates both but forces an explicit cross-region consistency trade-off.
- RTO and RPO are the specific, deliberately-chosen numbers that determine which architecture and replication strategy a given system actually needs — set by weighing cost against business impact, not defaulted to "as strong as possible."
