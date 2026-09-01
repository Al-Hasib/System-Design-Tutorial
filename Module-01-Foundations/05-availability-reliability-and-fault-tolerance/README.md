# Availability, Reliability, Redundancy & Fault Tolerance

**Difficulty:** Beginner
**Estimated video length:** 12-15 min
**Prerequisites:** [04 - Scalability Basics: Vertical vs Horizontal Scaling](../04-scalability-basics-vertical-vs-horizontal-scaling/README.md), [02 - Functional vs Non-Functional Requirements](../02-functional-vs-non-functional-requirements/README.md)

## Learning Objectives

- Define availability and reliability precisely, and explain how they differ.
- Understand what "the nines" (99.9%, 99.99%, etc.) mean in practice, in terms of actual downtime.
- Define redundancy and fault tolerance, and explain how they enable high availability.
- Recognize the concept of a single point of failure and how to eliminate one.
- Understand basic strategies for building fault-tolerant systems: replication, failover, and health checks.

## Script

### Hook / Intro

Every big tech company loves to advertise "99.99% uptime." That number sounds impressive, but what does it actually mean in practice — and more importantly, how do engineering teams actually achieve it? In this video, we're closing out Module 1 by covering the vocabulary of reliability: availability, redundancy, and fault tolerance. These concepts directly build on what we just learned about horizontal scaling, because it turns out that adding more machines doesn't just help you handle more traffic — it's also one of your most powerful tools for surviving failure.

### Availability vs Reliability

Let's start by distinguishing two words people often use interchangeably but which mean different things.

**Availability** is the percentage of time a system is up and able to respond to requests, measured over some period (usually a year). It's typically expressed as a percentage, like 99.9% or 99.99% — often nicknamed "the nines."

**Reliability** is about a system performing its intended function *correctly*, consistently, without failure, over a given period of time. A system can be available — technically up and responding — while still being unreliable, if it's returning wrong answers, corrupting data, or randomly timing out on some requests. Availability asks "is it up?" Reliability asks "does it work correctly when it's up?"

Here's a way to remember the difference: a server that's running but returning error pages to half its requests is *available* in the technical sense — it's responding — but it is not *reliable*, because it's not actually doing its job correctly.

### Understanding "The Nines"

Let's make "99.99% availability" concrete by translating it into actual downtime, because the difference between the nines is much bigger than it looks:

- **99% availability** ("two nines") = about 3.65 days of downtime per year
- **99.9% availability** ("three nines") = about 8.76 hours of downtime per year
- **99.99% availability** ("four nines") = about 52.6 minutes of downtime per year
- **99.999% availability** ("five nines") = about 5.26 minutes of downtime per year

Notice how each additional nine is roughly a 10x reduction in allowed downtime. Going from three nines to four nines doesn't just mean "a little better" — it means engineering the system to be ten times more resilient, which typically requires fundamentally more sophisticated infrastructure: redundancy across multiple servers, multiple data centers, automated failover, and rigorous monitoring. This is why five-nines systems — think telecom infrastructure or certain financial systems — are so expensive and complex to build and operate.

### Redundancy: The Core Tool for High Availability

So how do you actually achieve high availability? The primary technique is **redundancy** — having backup components ready to take over if a primary component fails. Instead of relying on one server, one database, or one data center, you deploy multiples of each, so that the failure of any single one doesn't bring the whole system down.

This directly connects to what we covered in the last video: horizontal scaling, where you run multiple servers instead of one, naturally gives you redundancy as a side effect. If one server in a pool of ten crashes, the other nine keep serving traffic without users even noticing — assuming your load balancer detects the failure and stops routing traffic to the dead server, which we'll cover more in Module 2.

The opposite of redundancy is a **single point of failure (SPOF)** — any one component whose failure takes down the entire system. A classic SPOF is a single database server with no backup: if it goes down, every part of your application that needs data grinds to a halt, even if your web servers are perfectly healthy. Identifying and eliminating single points of failure is one of the most important habits in reliable system design — for every critical component, ask yourself: "what happens if this one thing dies right now?"

### Fault Tolerance

**Fault tolerance** is a system's ability to continue operating correctly even when some of its components fail. Notice this is a stronger claim than just "having redundancy" — fault tolerance means failures are actually handled gracefully and automatically, ideally without any human intervention or user-visible disruption.

A few standard techniques make fault tolerance real, rather than just theoretical:

- **Replication** — keeping multiple copies of data or services, so if one copy is lost, another is available (covered deeply in Module 3).
- **Failover** — automatically switching traffic from a failed component to a healthy backup, often triggered by a **health check** — a periodic automated probe that asks "are you still working?" and removes unhealthy instances from rotation if they stop responding correctly.
- **Graceful degradation** — designing a system so that when a non-critical component fails, the system still functions, just with reduced functionality, rather than failing completely. For example, if a recommendation engine goes down, an e-commerce site might just stop showing "recommended for you" sections while checkout and browsing continue to work fine.

### Real-World Example

Consider a major streaming service. It doesn't run on one server, or even one data center — it runs across multiple geographically distributed data centers. If an entire data center goes offline due to a power outage or natural disaster, traffic automatically fails over to another region, and most users never even notice. Internally, its databases are replicated across multiple machines, so losing one database server doesn't lose any data. Health checks continuously probe every service, automatically routing traffic away from any instance that starts failing. This is fault tolerance and redundancy working together at massive scale — and notice that none of it is possible with a single vertically-scaled machine. It requires the horizontal scaling foundation from the previous video.

### Recap

Let's recap Module 1's final video. Availability measures the percentage of time a system is up and responding; reliability measures whether it performs correctly when it is up — they're related but distinct. "The nines" quantify availability targets, and each additional nine represents roughly a 10x reduction in acceptable downtime, requiring proportionally more sophisticated infrastructure. Redundancy — having backup components — is the core tool for achieving high availability, and eliminating single points of failure is essential. Fault tolerance takes this further: automatically detecting and recovering from failures via replication, failover, and health checks, so the system keeps working correctly even when individual parts break.

### What's Next

That wraps up Module 1: Foundations. You now understand what system design is, how to gather and classify requirements, how the internet's request pipeline works, and the two central pillars of every large system — scalability and reliability. In Module 2, we build directly on this foundation, going deep into networking and communication: HTTP and REST APIs in detail, load balancing (which we've referenced several times already), reverse proxies, API gateways, and real-time communication with WebSockets. See you in Module 2.

## Key Takeaways

- Availability = percentage of time a system is up and responsive; reliability = whether it performs its function correctly and consistently when up. They are related but different.
- "The nines" (99%, 99.9%, 99.99%, 99.999%) each represent roughly a 10x reduction in allowed annual downtime — going from three to four nines is a major engineering leap, not a minor improvement.
- Redundancy (backup components) is the primary tool for achieving high availability; a single point of failure (SPOF) is any component whose failure takes down the whole system.
- Fault tolerance means a system continues to operate correctly through failures automatically, via techniques like replication, failover with health checks, and graceful degradation.
- Horizontal scaling (from the previous video) provides redundancy as a natural side effect, connecting scalability and reliability as two sides of the same architectural approach.
