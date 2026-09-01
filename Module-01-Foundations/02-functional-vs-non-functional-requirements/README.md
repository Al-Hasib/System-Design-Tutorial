# Functional vs Non-Functional Requirements

**Difficulty:** Beginner
**Estimated video length:** 10-14 min
**Prerequisites:** [01 - What is System Design?](../01-what-is-system-design/README.md)

## Learning Objectives

- Define functional requirements and non-functional requirements and clearly distinguish between them.
- Practice extracting both types of requirements from an ambiguous prompt.
- Understand common non-functional categories: performance, scalability, availability, consistency, security, and cost.
- Learn how to estimate scale using back-of-the-envelope calculations (users, requests per second, storage).
- Appreciate why skipping requirements gathering leads to bad designs and failed interviews.

## Script

### Hook / Intro

Imagine an interviewer says: "Design Instagram." If your very next move is to draw a database and a server, you've already made a mistake — and it's the single most common mistake in system design interviews. Before you design anything, you need to know *what* you're actually building and *how well* it needs to perform. That's what requirements gathering is for, and it splits into two buckets: functional and non-functional requirements. Let's break both down.

### What Are Functional Requirements?

Functional requirements describe **what the system should do** — the actual features and behaviors from a user's perspective. They answer the question: "If I use this product, what can I do with it?"

For Instagram, functional requirements might include: users can upload photos and videos, users can follow other users, users can see a feed of posts from people they follow, users can like and comment on posts, users can search for other users or hashtags.

Notice these are all things a product manager would write in a feature spec. They're concrete, testable, and directly visible to the end user. If a functional requirement is missing or wrong, the product simply doesn't do what it's supposed to do.

### What Are Non-Functional Requirements?

Non-functional requirements describe **how well the system does what it does** — the quality attributes and constraints under which those features must operate. They don't add new features; they define the standards those features must meet.

Common categories of non-functional requirements include:

- **Performance/Latency** — How fast must a response come back? (e.g., feed loads in under 200ms)
- **Scalability** — How many users, and how much traffic, must the system handle, now and in the future?
- **Availability** — What percentage of the time must the system be up and reachable? (e.g., 99.9% uptime)
- **Consistency** — When data changes, how quickly must all parts of the system see that change?
- **Durability** — Once data is saved, can it ever be lost?
- **Security** — What data needs to be protected, and from whom?
- **Cost** — What's the budget for infrastructure?

For Instagram, non-functional requirements might look like: the system must support 500 million daily active users, feed loads must render in under 300 milliseconds, the platform must be available 99.99% of the time, uploaded photos must never be lost once confirmed, and the read-to-write ratio is heavily skewed toward reads (people view far more posts than they upload).

### Why the Distinction Matters

Here's the key insight: **functional requirements tell you what to build; non-functional requirements tell you how to build it.** Two systems with identical functional requirements — say, "users can post short messages" — can end up architecturally completely different if one needs to support 10 users and the other needs to support 500 million. The feature is the same. The engineering is not.

This is also exactly why interviewers deliberately give you vague prompts like "design Twitter." They want to see if you'll pause and ask clarifying questions rather than assuming. A strong candidate says: "Before I design this, let me clarify a few things — roughly how many users are we targeting? What's the read versus write ratio? Do we need strong consistency, or is eventual consistency acceptable? Is this a global user base needing multi-region support?" These questions directly shape your architecture — your choice of database, whether you need a cache, whether you need multiple data centers, and so on.

### How to Estimate Scale (Back-of-the-Envelope)

A practical skill here is doing rough capacity estimation. You don't need exact numbers — you need order-of-magnitude estimates that inform your design.

Say you're told the app has 100 million daily active users, and on average each user makes 10 requests per day. That's 1 billion requests per day. Divide by roughly 86,400 seconds in a day, and you get about 11,600 requests per second on average. But traffic isn't evenly distributed — there are peak hours — so you might multiply by 2 to 3x to estimate peak load, landing around 25,000-35,000 requests per second at peak. That single calculation already tells you: this is not a system that runs on one server. You need load balancing and horizontal scaling from day one — concepts we'll cover in upcoming videos.

The same logic applies to storage: if 10 million photos are uploaded per day, and each photo averages 2MB, that's 20TB of new data every single day. That number alone should tell you that you can't just dump files into a single database — you'll need distributed storage and probably a CDN, both of which we cover in Module 4.

### Real-World Example

Let's contrast two real systems briefly. A **URL shortener** like bit.ly has functional requirements like "shorten a URL" and "redirect a short URL to the original." Its non-functional requirements emphasize extremely low read latency (redirects must be nearly instant) and a massive read-to-write ratio, since a URL is created once but clicked thousands of times. Compare that to a **banking system**, where the functional requirement "transfer money between accounts" comes with a non-functional requirement of strict consistency and durability — you absolutely cannot lose or double-count a transaction, even if that means sacrificing some speed. Same general shape (client, server, database) — wildly different priorities because the non-functional requirements differ.

### Recap

To recap: functional requirements define *what* the system does — the visible features. Non-functional requirements define *how well* it must do them — performance, scale, availability, consistency, security, cost. Always gather both before proposing an architecture, and use rough back-of-the-envelope math to turn vague scale descriptions ("lots of users") into concrete numbers that actually drive design decisions.

### What's Next

Now that we know how to figure out *what* to build and *how well* it needs to perform, the next video steps back to the plumbing underneath everything: client-server architecture and how the internet actually works — DNS, IP addresses, TCP/IP, and HTTP. Every request your system handles travels through this pipeline, so understanding it is essential before we talk about load balancers, APIs, or databases. See you there.

## Key Takeaways

- Functional requirements = what the system does (features, user-visible behavior).
- Non-functional requirements = how well the system does it (performance, scalability, availability, consistency, durability, security, cost).
- Always clarify both types of requirements before designing — this is the first move in any real design process or interview.
- Back-of-the-envelope estimation (requests/second, storage/day) turns vague scale into numbers that directly drive architecture decisions.
- Identical functional requirements can lead to very different architectures depending on non-functional constraints.
