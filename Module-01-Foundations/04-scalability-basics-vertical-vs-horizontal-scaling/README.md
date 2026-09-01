# Scalability Basics: Vertical vs Horizontal Scaling

**Difficulty:** Beginner
**Estimated video length:** 12-15 min
**Prerequisites:** [03 - Client-Server Architecture & How the Internet Works](../03-client-server-architecture-and-how-the-internet-works/README.md), [02 - Functional vs Non-Functional Requirements](../02-functional-vs-non-functional-requirements/README.md)

## Learning Objectives

- Define scalability and explain why it's one of the central concerns in system design.
- Distinguish vertical scaling (scaling up) from horizontal scaling (scaling out).
- Understand the practical limits and trade-offs of each scaling strategy.
- Recognize why horizontal scaling introduces the need for load balancing and statelessness (previewing Module 2).
- Identify signals in a system that indicate it's time to scale, and which direction to scale in.

## Script

### Hook / Intro

In the last video, we traced a single request from your browser all the way to a server and back. But what happens when it's not one user making one request — it's a million users making a million requests, all at once? A single server, no matter how powerful, eventually can't keep up. This is the scalability problem, and there are exactly two fundamental ways to solve it: make your one server bigger, or add more servers. Let's dig into both.

### What is Scalability?

**Scalability** is a system's ability to handle increasing amounts of work — more users, more requests, more data — by adding resources, ideally without a drop in performance or a rewrite of the system. It's a non-functional requirement, something we introduced back in video 2, and it's one of the very first things you should think about once you know your expected scale.

Importantly, scalability isn't just about surviving today's load — it's about having a clear path to handle 10x or 100x that load in the future without a fundamental redesign. A system that works fine at 1,000 users but completely collapses at 10,000 users isn't scalable, even if it looks perfectly fine right now.

### Vertical Scaling: Scaling Up

The first strategy is **vertical scaling**, sometimes called "scaling up." This means adding more power to your *existing* machine — more CPU cores, more RAM, faster storage (like upgrading from a spinning hard disk to an SSD). Instead of adding new servers, you make the one server you have more powerful.

Think of it like upgrading from a compact car to a truck when you need to carry more cargo — same one vehicle, just a bigger, more capable version of it.

Vertical scaling has real advantages: it's simple. Your application code often doesn't need to change at all — the same single server just runs faster or handles more concurrent connections. There's no added complexity around coordinating multiple machines, no need to worry about how they communicate, and no data-consistency headaches between them.

But vertical scaling has a hard ceiling. There's a physical maximum to how much CPU, RAM, and storage a single machine can have — and once you hit it, you're stuck, no matter how much money you're willing to spend. Cloud providers do sell very large machines, but they get disproportionately expensive as you go up the tiers, and eventually the biggest machine on the market still isn't big enough for a company like Google or Netflix. There's also a reliability problem: a single powerful machine is still a **single point of failure**. If it goes down, your entire system goes down with it — a concept we'll explore fully in the next video on availability and fault tolerance.

### Horizontal Scaling: Scaling Out

The second strategy is **horizontal scaling**, or "scaling out." Instead of making one machine bigger, you add *more* machines, and spread the workload across all of them. Instead of one truck, you now have a fleet of vans, each carrying part of the load.

Horizontal scaling has essentially no hard ceiling — need more capacity? Add more machines. Companies like Google and Amazon run services across literally thousands or tens of thousands of individual servers, something that would be physically impossible with a single machine, no matter how large. It's also more resilient: if one machine in a fleet of a hundred fails, the other ninety-nine keep serving traffic, and the failed one can be replaced without anyone noticing.

But horizontal scaling introduces real complexity that vertical scaling doesn't have. First, you need a way to distribute incoming requests across all these machines — that's the job of a **load balancer**, which we'll cover in depth in Module 2. Second, your application typically needs to be **stateless**, meaning any server should be able to handle any request without depending on data stored only in that one server's memory from a previous request — otherwise, a user's second request might land on a different machine that doesn't know anything about their first request. Third, if your servers need to share data — like user sessions or application state — you now need a shared data store (a database or cache) that all servers can access, and coordinating data across multiple machines introduces its own set of challenges we'll spend significant time on throughout Modules 3 and 6.

### Choosing Between Them

In practice, most real systems use **both**, at different points. It's very common to start with vertical scaling because it's simple and cheap for the early stages of a product — just get a bigger server. But as demand grows, you inevitably hit that ceiling, and horizontal scaling becomes necessary for anything operating at real internet scale. A useful rule of thumb: vertical scaling buys you time; horizontal scaling buys you a future.

There's also a cost dimension. Beyond a certain point, doubling a single machine's specs doesn't just double its price — it can cost far more than double, because top-tier hardware carries a premium. Meanwhile, ten mid-tier machines can often be cheaper in total than one machine with ten times the power of a single mid-tier machine, while also giving you redundancy as a bonus.

### Real-World Example

Think about a small startup's first few months. They might run their entire application — web server and database — on a single reasonably-sized cloud instance. As their user base grows from hundreds to tens of thousands, they first respond by vertically scaling: upgrading to a bigger instance with more RAM and CPU. That works for a while. But once they're serving millions of users, no single machine can realistically handle that load or provide acceptable redundancy, so they shift to horizontal scaling: multiple web servers behind a load balancer, each stateless, all reading and writing to a shared, separately-scaled database tier. This exact pattern — vertical scaling first, horizontal scaling later — is one of the most common growth stories in the industry.

### Recap

To recap: scalability is a system's ability to handle growing load by adding resources. Vertical scaling means making one machine bigger — simple, but with a hard ceiling and a single point of failure. Horizontal scaling means adding more machines — effectively limitless and more resilient, but requiring load balancing, statelessness, and shared data stores. Most real systems use vertical scaling early and transition to horizontal scaling as they grow.

### What's Next

We've talked about scaling to handle *more* traffic — but what about handling *failures*? In the next video, we'll cover availability, reliability, redundancy, and fault tolerance: the vocabulary you need to describe how well a system stays up and running, even when individual pieces of it fail. This pairs directly with what we just learned, since horizontal scaling — adding more machines — turns out to be one of the best tools we have for building fault-tolerant systems too. See you there.

## Key Takeaways

- Scalability is a system's ability to handle growing load (users, requests, data) by adding resources, without a fundamental redesign.
- Vertical scaling (scaling up) = adding more power to an existing machine; simple but has a hard physical/cost ceiling and creates a single point of failure.
- Horizontal scaling (scaling out) = adding more machines; virtually limitless and more resilient, but requires load balancing, stateless application design, and shared data stores.
- Most systems start with vertical scaling for simplicity and move to horizontal scaling as they outgrow a single machine.
- Horizontal scaling is the foundation for both scalability and fault tolerance in large-scale systems.
