# Testing Distributed Systems: Load Testing & Chaos Engineering

**Difficulty:** Advanced
**Estimated length:** 14-18 min
**Prerequisites:** [43 - Observability: Logging, Metrics & Distributed Tracing](../43-observability-logging-metrics-and-distributed-tracing/README.md), [26 - Circuit Breaker, Retry & Bulkhead Patterns](../../Module-06-Distributed-Systems-Concepts/26-circuit-breaker-retry-and-bulkhead-patterns/README.md)

## Learning Objectives

- Explain why unit and integration tests don't tell you whether a system survives real production load or real infrastructure failure.
- Describe load testing, stress testing, and soak testing, and what distinct question each answers.
- Explain the philosophy behind chaos engineering: deliberately injecting failure to build confidence, not to prove nothing ever breaks.
- Describe the practice of running "game days" and the blast-radius discipline required to run chaos experiments safely.
- Design a basic resilience-testing plan for a system with known failure-handling mechanisms (retries, circuit breakers, redundancy).

## Script

### Hook / Intro

By this point in the course, we've designed systems with retries, circuit breakers, replication, redundancy, and failover (Module 6's resilience patterns). Here's the uncomfortable question almost nobody asks until an actual incident forces them to: how do you *know* any of that actually works? A unit test can verify your circuit breaker's state machine transitions correctly in isolation. It cannot tell you whether your circuit breaker's timeout is actually well-tuned for your real network's latency distribution, or whether your service actually degrades gracefully when a real downstream dependency really is slow, under real production traffic patterns. Today we cover the two disciplines that close this gap: load testing, which tells you how your system behaves under realistic (or extreme) traffic, and chaos engineering, which tells you how it behaves under realistic (or extreme) failure — deliberately, on your own schedule, instead of finding out for the first time during a real outage.

### Load Testing: Does It Survive Real Traffic?

**Load testing** means generating synthetic traffic against your system and observing how it behaves — but "load testing" is actually an umbrella term for several distinct questions. **Load testing** (in the narrow sense) checks behavior at expected peak traffic — does the system hold up at, say, the highest load you expect this Black Friday? **Stress testing** pushes traffic well beyond expected peak, specifically to find the breaking point and — just as important — to observe *how* the system fails: does it degrade gracefully (shedding low-priority work, returning 429s cleanly, per Module 6's rate limiting) or does it fail catastrophically (cascading failures taking down unrelated services, per Module 6's circuit breaker discussion)? **Soak testing** (or endurance testing) runs a sustained, moderate load for an extended period — hours or days — specifically to catch problems that only show up over time: memory leaks, gradual resource exhaustion (connection pools slowly leaking, disk filling with logs), or degradation that a short test simply wouldn't run long enough to reveal. Tools like k6, Locust, and Gatling are common for generating this synthetic load at scale, and the discipline of "test it before your users do" is exactly what separates finding your system's actual breaking point on a Tuesday afternoon from discovering it during a real, revenue-impacting traffic spike.

### Chaos Engineering: Does It Survive Real Failure?

Load testing answers "what happens under more traffic." **Chaos engineering** answers a different question: "what happens when a piece of my infrastructure actually breaks, right now, in production or a production-like environment?" The core philosophy, pioneered publicly by Netflix's Chaos Monkey, is genuinely counter-intuitive at first: deliberately and randomly cause failures — kill a server instance, introduce network latency, simulate a dependency timing out — in order to build *confidence* that your system's failure-handling mechanisms (the ones from Module 6: retries, circuit breakers, redundancy, failover) actually work as designed, under conditions you don't get to fully script in advance. The reasoning: your system is going to experience real failures eventually, whether or not you chose the timing. Chaos engineering just insists on choosing the timing — ideally during business hours, with the team watching dashboards and ready to intervene, rather than at 3am during an actual incident with a real customer impact clock running.

This is not "randomly break production and see what happens" recklessness — the actual practice is disciplined and incremental. You start with a **hypothesis**: "if the payments service's primary database replica fails, traffic should fail over to the secondary within 10 seconds with no more than a brief latency blip." You define a tightly bounded **blast radius**: run the experiment against a small percentage of traffic or a non-critical environment first, with an automatic "abort" trigger if key metrics (the ones from your observability setup) cross a danger threshold. You run the experiment, observe whether the actual behavior matched the hypothesis, and — this is the actual point — you almost always find something your design assumed would work but doesn't quite, in some specific, previously-invisible way. Then you fix it, and gradually expand the blast radius and the sophistication of experiments as confidence grows.

### Game Days

Many organizations formalize this into a **game day**: a scheduled, planned exercise where a team deliberately simulates a specific failure scenario — a region going down, a critical dependency becoming unavailable, a data corruption event — and practices the actual incident response, not just the system's automated recovery. This tests something automated chaos experiments alone can't: whether the *people* and *runbooks* (the documented procedures for responding to a known failure mode) actually work under simulated pressure, whether the on-call engineer can actually find the right dashboard in time, and whether the escalation process functions the way it's documented to. A system with perfect automated failover can still turn into a real incident if the humans who need to be involved don't know how to diagnose what's happening — game days close that gap directly.

### Real-World Example

Consider a payments system with redundant database replicas and an automated failover mechanism (Module 3's replication) — the design says failover should complete within 10 seconds. A chaos experiment tests this directly: during a scheduled window, with the team watching dashboards, the primary replica is deliberately killed, and the team observes what actually happens — maybe failover completes in 8 seconds as designed, or maybe it turns out the application's connection pool doesn't reconnect properly and needs a full restart, taking two full minutes instead — a real gap between designed and actual behavior that a design document alone would never have surfaced. Separately, a load test ahead of a known high-traffic sales event runs synthetic traffic at 3x last year's peak, revealing that a specific downstream inventory service starts timing out at 2.5x — well within this year's expected peak — giving the team weeks of runway to fix a real capacity problem instead of discovering it live during the actual event.

### Recap

Unit and integration tests verify logic in isolation; they don't tell you whether a system survives real production load or real infrastructure failure — that requires deliberately testing for both. Load testing (at expected peak), stress testing (well beyond peak, to find the breaking point and observe failure mode), and soak testing (sustained load over time, to catch slow leaks and gradual degradation) each answer a distinct question about traffic. Chaos engineering deliberately injects real failure — with a clear hypothesis and a disciplined, bounded blast radius — to build confidence that your resilience mechanisms actually work, rather than assuming they do because the design document says so. Game days extend this to test the humans and runbooks, not just the automated systems, closing the last gap between "designed to survive failure" and "actually survives failure, including the human response to it."

### What's Next

We've now deliberately tested a single system's resilience to failure. The last video in this module goes one level up: what happens when the failure isn't one server or one dependency, but an entire region — and how multi-region architecture and disaster recovery planning define exactly how bad that's allowed to get.

## Key Takeaways

- Unit and integration tests verify logic in isolation; they don't prove a system survives real production load or real infrastructure failure.
- Load testing checks behavior at expected peak traffic; stress testing pushes past peak to find the breaking point and observe failure mode; soak testing runs sustained load over time to catch slow leaks and gradual degradation.
- Chaos engineering deliberately injects failure — with a clear hypothesis and a disciplined, bounded blast radius — to build confidence that resilience mechanisms (retries, circuit breakers, failover) actually work under real conditions.
- Chaos experiments almost always reveal a gap between designed and actual behavior — that's the point, not a sign of a badly-run experiment.
- Game days test the humans and runbooks, not just the automated systems — a perfect automated failover can still become a real incident if the on-call response doesn't work.
