# Practice & Interview Questions

**1. Why don't unit and integration tests tell you whether a system survives real production load or real infrastructure failure?**
Unit and integration tests verify logic in isolation, under controlled, predictable conditions — they can confirm a circuit breaker's state machine transitions correctly, for example, but not whether its timeout is well-tuned for real network latency, or how the system actually behaves when a real downstream dependency is genuinely slow under real production traffic patterns.

**2. Distinguish load testing, stress testing, and soak testing.**
Load testing checks behavior at expected peak traffic. Stress testing pushes well beyond expected peak specifically to find the breaking point and observe the failure mode. Soak testing runs sustained, moderate load over an extended period to catch problems (like memory leaks or gradual resource exhaustion) that only appear over time, not in a short test.

**3. What does it mean for a system to "fail gracefully" versus "fail catastrophically" under stress testing, and give an example of each.**
Failing gracefully means the system sheds or rejects excess work in a controlled way — e.g., cleanly returning 429 Too Many Requests once past capacity (recall rate limiting from Module 6). Failing catastrophically means the overload cascades — e.g., one overwhelmed service causes timeouts that cascade into unrelated services failing too, a lack of the isolation circuit breakers and bulkheads are meant to provide.

**4. What is the core philosophy behind chaos engineering?**
Deliberately and proactively injecting real failure into a system, on your own schedule and with your team watching, to build confidence that resilience mechanisms (retries, circuit breakers, redundancy, failover) actually work — rather than assuming they work because the design document says so, and finding out otherwise for the first time during a real, uncontrolled incident.

**5. Describe the five (or six) steps of a disciplined chaos engineering experiment.**
Form a specific hypothesis about expected behavior under a failure, define a bounded blast radius (small traffic percentage or non-critical environment) with an abort trigger, run the experiment by injecting the real failure, observe whether actual behavior matched the hypothesis, fix whatever gap was found, and gradually expand the blast radius and sophistication of future experiments.

**6. Why is defining a "blast radius" before running a chaos experiment important?**
Without a bounded blast radius, an experiment could cause a real, uncontrolled outage affecting all users instead of a small, controlled test. Defining it upfront (a small traffic percentage, a non-critical environment, an automatic abort trigger tied to observability metrics) is what makes chaos engineering a disciplined practice rather than reckless "randomly break production" behavior.

**7. Why does a chaos experiment "almost always" reveal a gap between designed and actual behavior, and why is this considered valuable rather than a failure of the exercise?**
Real systems accumulate subtle behaviors — connection pool quirks, timing assumptions, configuration drift — that a design document doesn't capture and that only show up under actual failure conditions. Finding these gaps deliberately, during a scheduled, controlled experiment, is exactly the value of the exercise — it's far better to discover them there than during an uncontrolled real incident.

**8. What is a "game day," and what does it test that an automated chaos experiment alone doesn't?**
A game day is a scheduled exercise simulating a specific failure scenario where a team practices actual incident response — not just observing whether the automated system recovers, but whether the on-call engineer can find the right dashboard, follow the runbook correctly, and escalate appropriately under simulated pressure. It tests the humans and processes around the system, which a purely automated chaos experiment doesn't cover.

**9. Scenario: A team's design document says database failover should complete within 10 seconds, but this has never been tested in a realistic way. What would you recommend, and why?**
Run a chaos experiment with the specific hypothesis "failover completes within 10 seconds with minimal latency impact," in a bounded blast radius, deliberately killing the primary replica during a scheduled window while watching metrics — this directly tests whether the documented design assumption holds in practice, rather than continuing to assume it does until a real failure proves otherwise.

**10. True or False: Running a load test before a known high-traffic event (like a big sales day) is primarily about confirming the system works, and finding no problems is the best outcome.**
False, in spirit — the real value of a load test ahead of a known event is finding capacity problems (like a downstream service timing out below expected peak) while there's still time to fix them. A load test that finds nothing might mean the system is genuinely ready, but it's worth being skeptical of a test that never finds anything, since it may not be pushing hard enough or covering the right scenarios.
