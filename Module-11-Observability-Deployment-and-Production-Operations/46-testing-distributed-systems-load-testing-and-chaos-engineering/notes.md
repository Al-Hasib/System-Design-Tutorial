# Study Notes: Testing Distributed Systems

## Definitions

- **Load testing:** Generating synthetic traffic at expected peak levels to verify the system holds up.
- **Stress testing:** Pushing traffic well beyond expected peak to find the breaking point and observe the failure mode (graceful degradation vs. cascading failure).
- **Soak testing (endurance testing):** Running sustained, moderate load over an extended period to catch slow leaks and gradual degradation.
- **Chaos engineering:** Deliberately injecting real failures into a system to build confidence that its resilience mechanisms actually work.
- **Blast radius:** The bounded scope (percentage of traffic, specific environment) an experiment is allowed to affect.
- **Game day:** A scheduled exercise simulating a specific failure scenario to test both automated recovery and human incident response.

## Types of Load Testing

| Type | Traffic level | Question answered |
|---|---|---|
| Load test | Expected peak | Does it hold up at the traffic we actually expect? |
| Stress test | Well beyond peak | Where's the breaking point, and does it fail gracefully or catastrophically? |
| Soak test | Moderate, sustained over hours/days | Does it degrade slowly over time (leaks, resource exhaustion)? |

Common tools: k6, Locust, Gatling, Apache JMeter.

## Chaos Engineering Practice

1. **Form a hypothesis:** e.g., "if the primary DB replica fails, failover completes within 10 seconds with minimal latency impact."
2. **Define blast radius:** small percentage of traffic, or a non-critical/staging environment first; set an automatic abort trigger tied to observability metrics.
3. **Run the experiment:** inject the failure (kill an instance, add network latency, simulate a timeout) during a scheduled window with the team watching.
4. **Observe:** does actual behavior match the hypothesis? (It very often doesn't, in some specific way — that's the value of the exercise.)
5. **Fix and expand:** address what was found, then gradually increase blast radius/sophistication of future experiments.

## Load Testing vs. Chaos Engineering

| | Load Testing | Chaos Engineering |
|---|---|---|
| Tests | Behavior under more/sustained traffic | Behavior under real infrastructure failure |
| Answers | Does capacity/scaling hold up? | Do resilience mechanisms (retries, failover, circuit breakers) actually work? |
| Typical tools | k6, Locust, Gatling | Chaos Monkey, Gremlin, Litmus |

## Game Days

- Tests the humans and runbooks, not just automated systems.
- Simulates a specific scenario (region outage, dependency failure, data corruption) in a scheduled, planned exercise.
- Reveals gaps like: can the on-call engineer find the right dashboard, does the escalation process work, is the runbook actually accurate/current.

## Key Numbers / Facts

- Netflix's Chaos Monkey (part of the broader "Simian Army") was one of the first widely-publicized production chaos engineering tools, introduced around 2011.
- The Principles of Chaos Engineering (chaosprinciples.org) formalize the hypothesis-driven, blast-radius-controlled approach as an industry discipline.

## Summary

- Unit/integration tests verify logic in isolation; load testing and chaos engineering verify behavior under real traffic and real failure, respectively.
- Load/stress/soak testing each answer a distinct question about traffic behavior; chaos engineering answers whether resilience mechanisms actually work under real, deliberately-injected failure.
- Game days extend this to test human response and runbooks — a perfect automated design can still fail if the incident response around it doesn't work.
