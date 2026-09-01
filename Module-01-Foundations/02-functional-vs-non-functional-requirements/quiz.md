# Practice & Interview Questions

1. **Define functional requirements and give two examples for a ride-sharing app.**
   Functional requirements describe what the system does. Examples: a rider can request a trip; a driver can accept or decline a trip request.

2. **Define non-functional requirements and give two examples for the same ride-sharing app.**
   Non-functional requirements describe how well the system performs its functions. Examples: trip matching must complete in under 5 seconds; the system must be available 99.9% of the time.

3. **Why do interviewers deliberately give vague prompts like "design Twitter"?**
   To see whether the candidate will ask clarifying questions about scale, read/write ratio, and consistency needs before designing, rather than making unfounded assumptions — this mirrors real-world ambiguity in product requirements.

4. **Scenario: You're asked to design a chat application. What are three clarifying questions you'd ask before designing?**
   Any three of: How many concurrent users/DAU? Is message delivery required in real time or can there be delay? Do we need message history/persistence? Group chats or 1:1 only? What consistency guarantees are needed (e.g., message ordering)?

5. **Walk through a back-of-the-envelope calculation for a system with 50 million DAU, each making 20 requests/day.**
   50M × 20 = 1B requests/day. 1B / 86,400 ≈ 11,600 RPS average. Multiply by ~2-3x for peak → roughly 25,000-35,000 RPS at peak.

6. **Why is the read-to-write ratio an important non-functional consideration?**
   It directly influences architecture choices — e.g., a read-heavy system benefits heavily from caching and read replicas, while a write-heavy system needs a design optimized for fast, durable writes and possibly sharding.

7. **Give an example of two systems with the same functional requirement but different non-functional requirements, resulting in different architectures.**
   A personal to-do list app and a global banking ledger might both have the functional requirement "record a transaction/task," but the banking system needs strict consistency and durability guarantees the to-do app does not, leading to very different database and architecture choices.

8. **What does "durability" mean as a non-functional requirement, and how is it different from "availability"?**
   Durability means once data is successfully saved, it will not be lost, even during failures. Availability means the system can be reached and respond to requests at a given rate of uptime. A system can be available but not durable (e.g., in-memory only storage) or durable but temporarily unavailable.

9. **Scenario: A stakeholder says "we need the system to be fast." What follow-up question turns this into an actionable non-functional requirement?**
   Ask for a specific target, e.g., "What's the maximum acceptable response time — 100ms, 500ms, 2 seconds? For which operations specifically?" Vague quality goals must be converted into measurable thresholds.

10. **Why should capacity estimation use rough, order-of-magnitude numbers rather than precise calculations?**
    Because early in design, exact figures aren't available or necessary — the goal is to identify whether the system needs one server or thousands, not to compute an exact number; rough estimates are sufficient to guide major architecture decisions.

11. **What's a potential consequence of skipping requirements gathering and jumping straight into architecture?**
    You may build a system that's over-engineered (wasting time/cost) or under-engineered (unable to handle real load or reliability needs), and in an interview setting, you risk being seen as someone who doesn't validate assumptions before acting.

12. **Categorize each as functional (F) or non-functional (NF): (a) "Users can reset their password," (b) "99.95% uptime," (c) "Search results return in under 200ms," (d) "Users can filter search results by date."**
    (a) F, (b) NF, (c) NF, (d) F.
