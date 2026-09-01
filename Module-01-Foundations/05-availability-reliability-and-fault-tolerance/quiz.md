# Practice & Interview Questions

1. **Define availability and reliability, and explain the difference between them.**
   Availability is the percentage of time a system is up and able to respond to requests. Reliability is whether the system performs its intended function correctly and consistently. A system can be available (responding) while unreliable (returning wrong results or errors).

2. **How much downtime per year does 99.9% availability allow? What about 99.99%?**
   99.9% allows about 8.76 hours of downtime per year. 99.99% allows about 52.6 minutes per year — roughly 10x less.

3. **What is a single point of failure (SPOF)? Give an example.**
   A SPOF is any single component whose failure takes down the entire system. Example: a single database server with no replica — if it fails, the whole application loses access to data.

4. **What is redundancy, and how does it help achieve high availability?**
   Redundancy means having backup/duplicate components (servers, databases, data centers) so that the failure of any one doesn't take down the whole system — traffic or data access shifts to a healthy backup instead.

5. **Define fault tolerance and explain how it differs from simply "having redundancy."**
   Fault tolerance is a system's ability to automatically continue operating correctly despite component failures. Redundancy (having backups) is a necessary building block, but fault tolerance also requires automatic detection (health checks) and automatic recovery (failover) — redundancy without automated failover still requires manual intervention.

6. **What is a health check, and what role does it play in failover?**
   A health check is a periodic automated probe that verifies whether a service instance is working correctly. When a health check fails repeatedly, the system (e.g., a load balancer) removes that instance from rotation and routes traffic to healthy instances instead — enabling automatic failover.

7. **What is graceful degradation? Give an example.**
   Graceful degradation is designing a system so that when a non-critical component fails, the system keeps working with reduced functionality rather than failing entirely. Example: an e-commerce site's recommendation engine goes down, but users can still browse and check out normally, just without "recommended for you" sections.

8. **Scenario: Your system has 99.99% availability but users frequently report getting incorrect search results. Is this an availability problem or a reliability problem?**
   It's a reliability problem — the system is up and responding (available), but it isn't performing its function correctly, which is a reliability concern rather than an availability one.

9. **Why does horizontal scaling (from the previous video) naturally improve fault tolerance?**
   Because running multiple servers instead of one means the failure of any single server doesn't take down the whole system — the remaining healthy servers continue serving traffic, provided a load balancer detects and routes around the failed one.

10. **Scenario: A company wants to move from 99.9% to 99.999% availability. Why is this a much bigger engineering challenge than the percentage difference suggests?**
    Because each additional nine represents roughly a 10x reduction in allowed downtime — going from three nines (~8.76 hrs/year) to five nines (~5.26 min/year) requires eliminating nearly all single points of failure across servers, databases, networks, and even data centers, plus highly automated failover, which is a dramatic increase in engineering complexity and cost.

11. **What is replication, and which future module will cover it in depth?**
    Replication is keeping multiple copies of data (or services) so that losing one copy doesn't lose the data. It's a core fault-tolerance technique and will be covered in depth in Module 3 (Databases & Storage), specifically in the database replication video.

12. **True or False: A system with 100% availability is achievable and should be the goal for all systems.**
    False — 100% availability is practically unachievable (and extremely costly to approach) due to hardware failures, network issues, and maintenance needs; real systems set a target based on business needs (e.g., 99.9% or 99.99%) and balance cost against acceptable downtime.
