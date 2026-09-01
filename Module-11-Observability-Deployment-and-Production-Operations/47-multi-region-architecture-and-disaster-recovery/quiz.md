# Practice & Interview Questions

**1. Why isn't redundancy within a single region enough protection for a critical system?**
Redundancy within a region (multiple servers, availability zones, replicated databases) protects against machine, rack, or even single-data-center failures within that region. It does nothing if the entire region becomes unavailable — due to a power outage, networking failure, or disaster affecting shared regional infrastructure — which is a rarer but real risk that only multi-region architecture addresses.

**2. Describe active-passive multi-region architecture and its two main costs.**
One region (active) serves all live traffic while a second region (passive/standby) keeps a replicated copy of data but doesn't serve production traffic under normal operation; if the active region fails, traffic fails over to the passive region. The two costs are idle capacity (the standby region is paid for but mostly unused) and failover delay (redirecting traffic, warming up, verifying data currency all take real time).

**3. How does active-active architecture eliminate both of active-passive's main costs, and what cost does it introduce instead?**
Every region serves live traffic simultaneously, so there's no idle standby capacity and no failover delay (no single active region needs to "take over" from). The cost it introduces is cross-region data consistency: since writes can be accepted in more than one region at once, the system must confront the CAP/PACELC trade-off directly on every write.

**4. Define RTO and RPO, and explain why they're described as business decisions rather than purely technical ones.**
RTO (Recovery Time Objective) is the maximum acceptable time between a disaster and the system being back online. RPO (Recovery Point Objective) is the maximum acceptable amount of data loss, measured in time. They're business decisions because the "correct" values depend on weighing the cost of stronger guarantees (latency, infrastructure, engineering complexity) against the actual business cost of downtime or data loss for that specific system — there's no universally correct number.

**5. How does an RPO of zero (no acceptable data loss) constrain the replication strategy a system can use?**
An RPO of zero requires synchronous cross-region replication — every write must be confirmed as replicated to another region before being considered complete — which adds real latency to every write, proportional to the network distance/conditions between regions, in exchange for the guarantee that no confirmed write can ever be lost even if a region fails immediately after.

**6. Why would active-active architecture typically be required to meet an RTO of just a few seconds?**
Any architecture involving failover — redirecting traffic from one region to another — takes some real time (DNS propagation, application warm-up, verification), which realistically can't be reduced to just seconds. Active-active avoids this problem entirely because there's no single active region that needs to be failed over from in the first place; other regions are already serving traffic.

**7. Why might two different systems within the same company reasonably have very different RTO/RPO targets?**
Because the business cost of downtime and data loss differs by system — an internal analytics dashboard has a low cost of being down for hours or losing a day of data, while a customer-facing payment system has a very high cost for even a minute of downtime or any lost transaction. RTO/RPO should reflect each system's actual business impact, not a single company-wide default.

**8. Scenario: A company needs zero data loss and near-instant recovery for its core transaction database, but wants to minimize the added write latency this requires. Is this achievable, and what would you tell them?**
This is a direct trade-off, not something to be optimized away — zero data loss (RPO zero) requires synchronous cross-region replication, which inherently adds latency to every write proportional to inter-region network conditions. You can only reduce that latency by choosing geographically closer regions or accepting a non-zero (but small) RPO via asynchronous replication — you can't have zero RPO with zero added latency; that's the actual trade-off being made.

**9. Scenario: An internal reporting tool used by a handful of analysts goes down for a few hours after a rare regional outage. Is a multi-region active-active architecture clearly justified here?**
Not necessarily — given the low business cost of a few hours of downtime for an internal tool with limited users, a documented manual recovery process from backups (a high RTO/RPO tolerance) may be entirely acceptable, and the added cost and complexity of active-active multi-region architecture likely isn't justified for this specific system.

**10. True or False: Active-active architecture avoids the CAP theorem trade-offs that single-region systems have to make.**
False. Active-active architecture doesn't avoid CAP/PACELC trade-offs — it makes them unavoidable and explicit, because writes can now be accepted concurrently in more than one region, forcing a direct choice between strong cross-region consistency (with its latency/availability cost) and eventual consistency (with its conflict-resolution requirement), which single-region systems don't have to confront at the region level at all.
