# Practice & Interview Questions

1. **What is system design, in your own words?**
   The process of defining how the components of a software system (clients, servers, databases, caches, queues, etc.) fit together and interact to satisfy requirements around scale, performance, and reliability — as opposed to writing the implementation of any single component.

2. **How is system design different from writing an algorithm or a coding interview problem?**
   Algorithm/coding problems usually have one optimal, verifiable answer with clear inputs/outputs. System design problems are open-ended, require clarifying ambiguous requirements, and are judged on the quality of trade-offs and reasoning rather than a single "correct" solution.

3. **Why do companies test system design skills in interviews?**
   Because real engineering work — especially at senior levels — requires deciding how features and services should be architected, not just implementing well-specified tickets. The interview simulates that decision-making process.

4. **Name three core building blocks common to most large-scale systems.**
   Any three of: client, server, database, cache, load balancer, message queue.

5. **Using the bricklayer/architect analogy, explain the difference between coding and system design.**
   A bricklayer focuses on laying one brick (one function/module) correctly. An architect decides the overall structure — how rooms connect, how the building withstands stress, how utilities flow — analogous to deciding how services, data stores, and communication patterns fit together in a system.

6. **Why is there rarely a single "correct" answer in system design?**
   Because every design choice involves trade-offs (cost, complexity, latency, consistency, availability) that depend on requirements and constraints, which vary between scenarios — a good design for one context can be wrong for another.

7. **Scenario: Your interviewer asks you to "design a URL shortener." What should be your very first step, before drawing any boxes?**
   Clarify requirements — expected scale (URLs shortened per day, read/write ratio), whether URLs expire, whether analytics are needed, etc. — before proposing an architecture (this is expanded in the next video).

8. **What are the two main reasons this course gives for learning system design?**
   (1) It's a standard part of technical interviews for mid/senior roles, and (2) it's a core real-world skill for engineers who need to make architecture decisions as they grow in their careers.

9. **List the 8 modules of this course in order.**
   Foundations; Networking & Communication; Databases & Storage; Caching & Content Delivery; Messaging & Asynchronous Systems; Distributed Systems Concepts; Architecture Patterns; Case Studies/Interview Practice.

10. **True or False: System design requires knowing a specific programming language deeply.**
    False — system design is largely language-agnostic; it's about architecture and trade-offs, though implementation knowledge can help you reason about feasibility.

11. **Scenario: A junior engineer says "I don't need system design, I just need to write good code." How would you respond?**
    Writing good code is necessary but not sufficient — as systems grow, decisions about how components interact, scale, and stay reliable become the dominant factor in success, and those decisions are exactly what system design teaches you to reason about.

12. **Why does this course start with "Foundations" before networking, databases, or distributed systems?**
    Because concepts like requirements gathering, client-server communication, scalability, and reliability are prerequisites referenced throughout every later module — starting elsewhere would require constantly backtracking to explain basic vocabulary.
