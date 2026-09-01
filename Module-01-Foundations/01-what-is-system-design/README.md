# What is System Design? Roadmap & How to Learn It

**Difficulty:** Beginner
**Estimated video length:** 10-12 min
**Prerequisites:** None — this is the first video in the course.

## Learning Objectives

- Define "system design" in plain language and distinguish it from coding/algorithms.
- Understand why system design matters for both interviews and real-world engineering careers.
- Recognize the major building blocks of modern systems (clients, servers, databases, caches, queues, load balancers) at a high level.
- Know the roadmap this course follows and how to study it effectively.
- Set realistic expectations for how deep "beginner" system design knowledge needs to go before moving to intermediate topics.

## Script

### Hook / Intro

Hey everyone, and welcome to the very first video in this System Design series. If you've ever opened a system design interview guide and felt immediately overwhelmed by terms like "sharding," "consistent hashing," or "CAP theorem," don't worry — that's exactly why this course exists. We're going to start from absolute zero and build up, one concept at a time, until those words feel as natural as "variable" or "loop" do in programming.

In this video, we're answering three questions: What actually *is* system design? Why should you care about it? And how is this course structured so you get the most out of it?

### What is System Design?

Let's start with a definition you can actually use. System design is the process of defining the architecture, components, modules, interfaces, and data flow of a software system to satisfy a specific set of requirements — things like how many users it needs to support, how fast it needs to respond, and how reliable it needs to be.

Notice what's missing from that definition: there's no mention of a specific programming language, a specific framework, or a specific line of code. That's the key mental shift. When you write code, you're usually solving "how do I implement this one function or feature correctly." When you do system design, you're stepping back and asking "how do all the pieces of this application fit together, and will that arrangement hold up under real-world conditions?"

Think of it like the difference between being a bricklayer and being an architect. A bricklayer is an expert at laying one brick perfectly. An architect decides where the walls go, how the building handles wind and earthquakes, how people move through the halls, and how the plumbing and electrical systems connect without interfering with each other. Both skills matter — but system design lives at the architect level.

### Why Does System Design Matter?

There are two big reasons to learn this, and they reinforce each other.

First, the interview reason. If you're interviewing for a mid-level or senior software engineering role at almost any tech company, you will very likely face a system design interview. Unlike a coding interview, where there's often one "correct" optimal solution, a system design interview is open-ended. The interviewer wants to see how you think: how you clarify requirements, how you make trade-offs, and how you communicate your reasoning. There's rarely a single right answer — there are better and worse trade-offs given the constraints.

Second, and honestly more important long-term, is the real-world engineering reason. As you grow in your career, you'll increasingly be asked not just to implement a ticket, but to decide *how* a feature or a whole product should be built. Should this be one service or three? Should we use a relational database or a NoSQL store? Do we need a cache? What happens when this feature gets 100x the traffic it has today? These are system design questions, and answering them well is what separates a junior engineer from a staff or principal engineer.

### The Building Blocks You'll Learn

Throughout this course, we'll progressively introduce the vocabulary and components that make up virtually every large-scale system you've ever used — think Twitter, Netflix, Uber, or Amazon. At a high level, these systems are built from:

- **Clients** — browsers, mobile apps, or other services that request something.
- **Servers** — machines that process those requests and return responses.
- **Databases** — where data is durably stored, structured, and queried.
- **Caches** — fast, temporary storage that reduces load on databases and speeds up responses.
- **Load balancers** — traffic directors that spread requests across multiple servers.
- **Message queues** — buffers that let different parts of a system communicate without being tightly coupled or immediately available at the same time.

Right now, these might just be names. That's completely fine. By the end of this course, you'll understand exactly what problem each of these solves and when to reach for it.

### How This Course is Structured

This course is organized into twelve modules, going from foundations all the way to full interview-style case studies:

We start here, in **Module 1: Foundations**, covering requirements gathering, how the internet actually works, and the core concepts of scalability and reliability. Then **Module 2** covers networking and communication patterns like HTTP, load balancing, and APIs. **Module 3** dives into databases and storage — SQL versus NoSQL, indexing, replication, sharding. **Module 4** covers caching and content delivery. **Module 5** introduces asynchronous systems: message queues and event-driven architecture. **Module 6** goes into deeper distributed systems concepts like consistent hashing, rate limiting, and consensus algorithms. **Module 7** covers architecture patterns like microservices versus monoliths. **Module 8** drops down a level to the transport protocols, web server internals, message formats, and security fundamentals that sit underneath everything we've covered so far. **Module 9** goes deeper still on database transaction internals and storage engines, plus GraphQL as an API alternative. **Module 10** covers distributed coordination and scale techniques — distributed locking, logical clocks, and probabilistic data structures. **Module 11** covers what it takes to actually run a system in production — observability, containers, safe deployments, resilience testing, and multi-region disaster recovery. And finally, **Module 12** ties everything together with full, realistic case studies — designing a URL shortener, a chat app, a news feed, and more.

### Real-World Example

Let's ground this with a quick example. Imagine you're asked to design "a URL shortener," like bit.ly. A pure coding approach might jump straight to writing a hash function. But a system design approach asks first: How many URLs do we need to shorten per day? How many redirects (reads) per second versus new URLs (writes) per second — because that ratio changes our architecture dramatically? Do old URLs ever expire? Do we need analytics on click counts? Only after answering these questions do you start choosing your database, your caching strategy, and your service architecture. We'll actually design this exact system together in Module 12, once you have all the building blocks.

### Recap

Let's recap. System design is about architecting how the pieces of a software system fit together to meet requirements around scale, speed, and reliability — it's architect-level thinking, not bricklayer-level thinking. It matters both because it's a standard part of technical interviews and because it's a core skill for growing as an engineer. This course moves through twelve modules, starting with foundations and ending with full case studies, and every module builds on the vocabulary from the one before it.

### What's Next

In the next video, we'll take the very first practical step of any real design process: learning how to gather and classify requirements into **functional** and **non-functional** requirements. This is the step almost everyone skips when they jump straight to whiteboarding boxes and arrows — and skipping it is exactly why so many designs fall apart under questioning. See you there.

## Key Takeaways

- System design focuses on how components (clients, servers, databases, caches, queues) fit together to meet requirements — not on writing individual lines of code.
- It matters for two reasons: technical interviews at most tech companies, and real-world career growth into senior/staff roles.
- There is rarely one "correct" design — only better or worse trade-offs given constraints.
- This course has 12 modules, progressing from foundational concepts to full system design case studies.
- Every later module builds directly on the vocabulary introduced in Module 1.
