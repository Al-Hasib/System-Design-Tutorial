# Practice & Interview Questions

1. **Define scalability in your own words.**
   Scalability is a system's ability to handle increasing load — more users, requests, or data — by adding resources, ideally without a performance drop or a full rewrite.

2. **What is vertical scaling? Give a concrete example.**
   Vertical scaling means adding more resources (CPU, RAM, storage) to an existing single machine — for example, upgrading a database server from 8GB to 64GB of RAM to handle more concurrent queries.

3. **What is horizontal scaling? Give a concrete example.**
   Horizontal scaling means adding more machines to handle the load collectively — for example, running 10 identical web server instances behind a load balancer instead of one large server.

4. **What is the main limitation of vertical scaling?**
   There's a hard physical ceiling on how much a single machine can be upgraded, cost grows disproportionately at the high end, and it remains a single point of failure regardless of how powerful it becomes.

5. **What two things does horizontal scaling typically require that vertical scaling does not?**
   A load balancer to distribute traffic across machines, and stateless application design (often paired with a shared external data store) so any server can handle any request.

6. **Why is a single, very powerful server considered a reliability risk even if it can handle the required load?**
   Because it's a single point of failure — if that one machine crashes or needs maintenance, the entire system becomes unavailable, regardless of how much spare capacity it had.

7. **Scenario: A startup's database server is at 90% CPU utilization during peak hours, but they expect steady, moderate growth over the next year. What's a reasonable short-term scaling response, and why?**
   Vertical scaling (upgrading to a larger instance) is often reasonable short-term — it's simple, requires no application changes, and buys time while the team plans for more complex horizontal scaling if growth continues.

8. **Scenario: A company's user base just grew 50x after a viral launch, and a single upgraded database server is still not enough. What should they consider, and what complexity does it introduce?**
   They should consider horizontal scaling — spreading load across multiple machines — which introduces the need for load balancing, stateless services, and a strategy for sharing/synchronizing data across those machines (e.g., replication or sharding, covered in Module 3).

9. **What does it mean for an application server to be "stateless," and why does it matter for horizontal scaling?**
   A stateless server doesn't rely on data stored only in its own memory from a previous request — any server can handle any request. This matters because a load balancer may route a user's requests to different servers over time, and none of them should need private, unshared knowledge of that user's prior interactions.

10. **Why might ten mid-tier servers sometimes be more cost-effective than one server with ten times the specs?**
    High-end hardware often carries a steep price premium relative to its performance gain (non-linear cost curve), so distributing the same total capacity across multiple mid-tier machines can be cheaper while also providing redundancy.

11. **True or False: Horizontal scaling has no practical limit on capacity.**
    Mostly true in principle — you can keep adding machines — though in practice you eventually hit new bottlenecks (e.g., database contention, network limits, coordination overhead) that require other techniques like sharding or caching, covered later in the course.

12. **Explain the "vertical first, horizontal later" pattern common in growing startups.**
    Early on, a company scales vertically because it's simple, fast, and requires no architecture changes — a bigger server is enough. As growth continues and a single machine's ceiling (capacity or reliability) is reached, they transition to horizontal scaling with multiple servers, a load balancer, and shared data stores to support continued and more resilient growth.
