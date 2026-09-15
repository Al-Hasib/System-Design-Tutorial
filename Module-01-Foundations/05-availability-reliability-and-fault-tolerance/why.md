# Why This Topic Matters: Availability, Reliability, Redundancy & Fault Tolerance

> **In one sentence:** At any meaningful scale, something is always broken — the question is never "how do we prevent failure" but "how do we keep working while failing."

## The World Before This Idea

The default mental model of a system is binary: it's up, or it's down. Under that model, the response to an outage is "find the bug, fix the bug, hope it doesn't happen again."

That model breaks down as soon as you have more than a handful of machines. A fleet of 200 servers with individually excellent hardware will still have a machine die most weeks. A dependency with 99.9% availability will be unavailable for roughly 43 minutes a month, and if your service calls five such dependencies synchronously, your ceiling is already below 99.5% — before you've written a single bug. Failure stops being an exception and becomes a background condition you design around.

## The Problems It Solves

### 1. One component failing takes down everything
**What you see:** The recommendations service — a nice-to-have — becomes slow. The product page calls it synchronously. Threads pile up waiting. Now the entire product page is down, because of a feature nobody would miss.

**Why it happens:** Without deliberate isolation, a dependency's availability becomes *your* availability. Synchronous coupling means their failure is your failure, and their latency is your latency.

**How fault tolerance solves it:** Timeouts, fallbacks, and graceful degradation let you serve the page without recommendations. The failure is contained to the feature that failed instead of cascading into the critical path.

### 2. Redundancy that doesn't actually help
**What you see:** You run three replicas across three servers and still have a full outage, because all three were in the same rack, on the same power feed, or behind the same load balancer — or because all three had the same bug and the same bad config pushed at the same time.

**Why it happens:** Redundancy only helps against *uncorrelated* failures. Copies that share a failure domain fail together.

**How this topic solves it:** It teaches you to think in failure domains — machine, rack, availability zone, region, config, code version — and to place redundancy across them, not within them.

### 3. Availability targets pulled out of thin air
**What you see:** A team commits to "five nines" in a design doc, with no idea that 99.999% means 5 minutes of downtime per year, which no human-operated deploy process can support.

**Why it happens:** Availability is discussed qualitatively ("highly available") instead of numerically, so nobody can tell whether the architecture matches the promise.

**How this topic solves it:** Translating nines into minutes makes the cost visible. 99.9% is 8.7 hours a year and is achievable with careful basics. 99.99% is 52 minutes and requires automated failover. 99.999% is 5 minutes and means no manual step can be on the critical path. Now the target is a budget you can spend and check.

### 4. Confusing "available" with "correct"
**What you see:** The health check is green, the dashboard is all up, and users are getting stale or wrong data.

**Why it happens:** Availability (does it respond?) and reliability (does it respond *correctly*, consistently, over time) are different properties. A service can be 100% available and 100% wrong.

**How this topic solves it:** Separating the two forces health checks and SLOs that measure real behavior — success rate, correctness, latency percentiles — rather than a process being alive.

## The Price You Pay

- **Cost.** Redundancy means paying for capacity you hope never to use. Multi-region active-active can roughly double infrastructure spend.
- **Complexity.** Failover, replication, and health checking are themselves systems that can fail — sometimes in more confusing ways than the failure they protect against. A flapping health check that ejects healthy nodes is a classic self-inflicted outage.
- **Consistency trade-offs.** Staying available during a partition generally means accepting stale or divergent data. You cannot have everything (see CAP).
- **Diminishing returns.** Going from 99% to 99.9% is often a few weeks of work. Going from 99.99% to 99.999% can consume an entire organization. Most products do not need it.

## When You Need It — and When You Don't

| Invest heavily when | Keep it simple when |
|---|---|
| Downtime costs money per minute (payments, ads, trading) | It's an internal tool with a tolerant audience |
| The system is safety- or compliance-critical | A batch job can simply be re-run tomorrow |
| You have enough scale that failures are routine | You have three servers and failures are rare events |
| Customers have contractual SLAs | You're pre-product-market-fit |

## Why This Shows Up in Interviews

Interviewers deliberately ask "what happens when this component dies?" at some point in nearly every design round. They're checking whether you treat your own diagram as an idealization or as something that runs on real hardware. Strong candidates volunteer failure modes unprompted: no single point of failure, health checks and failover, what degrades versus what breaks, and what the availability math on the whole chain actually works out to.

## How It Connects

This topic sets up a large fraction of the rest of the course. **Load balancers** need health checks to route around dead nodes. **Replication** exists so data survives a machine loss. **Circuit breakers, retries, and bulkheads** are the concrete implementations of fault isolation. **Multi-region architecture** is redundancy at the largest failure domain. **Chaos engineering** is how you prove any of it actually works.

**Next:** [HTTP/HTTPS & REST APIs Explained](../../Module-02-Networking-and-Communication/06-http-https-and-rest-apis/why.md) — the protocol layer all of this availability is delivered over.
